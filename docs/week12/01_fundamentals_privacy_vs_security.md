# Week 12 — Section 1: Fundamentals: Privacy vs Security



## Learning objectives

By the end of this topic, you should be able to:

- Explain the main ideas covered in **Week 12 — Section 1: Fundamentals: Privacy vs Security**.
- Connect these ideas to modern web application development.
- Recognize the patterns, terminology, and trade-offs used in practice.

---

## 1.1 Definitions & Scope

### What is Privacy?
- **Privacy** is about **who has access to information** — it concerns a person's right to control what data about them is collected, stored, shared, and used.
- It is fundamentally a **social and legal concept**, not just a technical one.
- Privacy = *the right of individuals to keep personal information to themselves* and decide what gets shared, with whom, and under what conditions.
- From a web development perspective: privacy means building systems where users are **not exposed to data collection or sharing beyond what they've consented to**.

### What is Security?
- **Security** is about **protecting data from unauthorized access** — the set of technical and procedural measures used to safeguard information from theft, tampering, loss, or disruption.
- Security = *mechanisms that prevent bad actors (and accidents) from accessing or corrupting data*.
- From a web development perspective: security means building systems that **enforce access controls, encrypt data, detect breaches, and resist attacks**.

### Key Similarities & Core Differences

| Aspect | Privacy | Security |
|---|---|---|
| **Goal** | Protect individuals | Protect data/systems |
| **Focus** | *Who* can access data | *How* to prevent unauthorized access |
| **Threat** | Overcollection, misuse, profiling | Hackers, breaches, malware |
| **Nature** | Legal, ethical, social | Technical, operational |
| **Compliance** | GDPR, HIPAA, CCPA | PCI-DSS, ISO 27001, NIST |
| **Violation can happen…** | …even without a breach (e.g., selling user data legally) | …even when privacy rules are followed (e.g., encrypted breach with minimal data) |

> **Key Insight:** You can have a **security breach without a privacy violation** (e.g., encrypted data stolen but unreadable), and you can have a **privacy violation without a security breach** (e.g., a company legally selling your data to brokers).

### Developer Implications & Responsibilities
- Developers are not just engineers — they are **stewards of user data**.
- **Legal liability:** Developers and organizations can be fined, sued, or prosecuted for both security failures and privacy violations.
- **Ethical responsibility:** Even when something is technically legal (e.g., selling anonymized data), it may violate user trust.
- Good practice: adopt a **privacy-by-design** and **security-by-default** mindset from the very beginning of a project.

---

## 1.2 Core Concepts of Privacy

### Personally Identifiable Information (PII)
- **PII** is any data that can be used to **identify a specific individual**, either alone or in combination with other data.
- **Direct PII examples:** Full name, National ID / Aadhaar / SSN, email address, phone number, physical address, passport number, biometrics (fingerprint, face).
- **Indirect / Quasi-identifiers:** Date of birth, ZIP code, gender — each alone may not identify someone, but **combined they often can**.
  - Classic study: 87% of Americans can be uniquely identified using only ZIP code + birth date + gender.
- **The Re-identification Problem:** "Anonymized" data is often not truly anonymous.
  - AOL released "anonymized" search logs in 2006 — reporters re-identified individuals within days using search queries alone.
  - Netflix Prize dataset (2006) — researchers re-identified users by cross-referencing with public IMDb reviews.

### User Rights & Control Over Personal Data
Under modern privacy frameworks (especially GDPR), users hold several core rights:

| Right | Description |
|---|---|
| **Right to Access** | Know what data is held about you |
| **Right to Rectification** | Correct inaccurate data |
| **Right to Erasure** | "Right to be Forgotten" — request deletion |
| **Right to Data Portability** | Receive data in a machine-readable format |
| **Right to Object** | Opt out of processing (e.g., marketing) |
| **Right to Restriction** | Limit how your data is used |

- **Informed Consent:** Data collection must be **explicit, informed, and revocable**.
  - Pre-ticked boxes or vague "by using this site you agree" clauses do **not** count as valid consent under GDPR.

### Privacy Mechanisms: Regulatory Mandates & EULAs
- **Regulatory Mandates:** Legally binding laws (GDPR, HIPAA, CCPA) that compel organizations to meet privacy standards, with financial penalties for non-compliance.
- **EULAs (End User License Agreements) & Privacy Policies:** Contractual documents that *disclose* how data is used — but frequently written to be incomprehensible, giving legal cover without genuine informed consent.
  - Studies show the average person would need **~76 working days per year** to read all the privacy policies they encounter.
  - EULAs are a **legal shield**, not a genuine privacy mechanism.
- **Key Difference:** Regulatory mandates are *enforced externally* (government bodies); EULAs are *agreed to by the user* (however uninformed).

### Evolution of Internet Privacy

#### The Early Web — "Nobody knows you're a dog" (1993)
- A famous *New Yorker* cartoon from 1993 captured the early internet's ethos: **anonymous by default**.
- The early web had no user accounts, no persistent tracking, no reliable identity — you were just an IP address, rarely logged meaningfully.
- Browsing was **stateless**: pages were served, and no memory of who you were was retained between sessions.

#### The Modern Web — Surveillance & Profiling (Now)
- The modern web is the **opposite** — you are profiled, tracked, and identified across virtually every website you visit.

**Key mechanisms of the modern surveillance web:**

| Mechanism | How it works |
|---|---|
| **Cookies & Supercookies** | Persistent identifiers stored in your browser, even across sessions |
| **Device Fingerprinting** | Browser, OS, screen resolution, fonts, timezone combined → near-unique identity, no cookies needed |
| **Tracking Pixels** | Invisible 1×1 images in emails/pages that report opens/views back to the sender |
| **Cross-Site Tracking** | Ad networks (Google, Meta) follow you across thousands of unrelated sites via embedded scripts |
| **Behavioral Profiling** | Purchase history, search queries, dwell time, scroll depth → targeted advertising and recommendations |

- **The Business Model of Surveillance:** The dominant web model is "free services in exchange for your data" — Google, Facebook, Instagram, TikTok are all free because *you are the product*.

---

## 1.3 Core Concepts of Security

### Data Safeguarding, Storage & Lifecycle Management
Security covers the **entire data lifecycle**, not just transmission:

| Stage | Security Concern |
|---|---|
| **Collection** | Secure channels (HTTPS/TLS), input validation |
| **Storage** | Encryption at rest, access controls, minimal privilege |
| **Processing** | Secure computation environments |
| **Transmission** | Encryption in transit (TLS 1.2+), certificate validation |
| **Archival** | Secure backups, offsite/offline copies |
| **Deletion** | Cryptographic erasure, secure wipe — data must be *truly* deleted |

### Implementation Measures

#### Secure Coding Practices
- Input validation & sanitization → prevent injection attacks.
- Output encoding → prevent XSS.
- Parameterized queries / ORM → prevent SQL injection.
- Avoid known insecure functions (`strcpy`, `gets` in C; `eval()` in JS/Python).
- Follow the **OWASP Top 10** as a baseline security checklist.

#### Vulnerability Patching
- Software has bugs; security bugs (CVEs — Common Vulnerabilities and Exposures) are discovered regularly.
- **Patch management:** Promptly apply security updates to OS, web server, frameworks, and dependencies.
- **Zero-day vulnerabilities:** Exploits with no patch yet — require compensating controls (WAF rules, disabling features, isolation).

#### Infrastructure Monitoring & Breach Detection
- **Intrusion Detection Systems (IDS):** Monitor network/system activity for suspicious patterns.
- **SIEM (Security Information and Event Management):** Aggregates logs, correlates events, alerts on anomalies.
- **Breach detection metrics:**
  - **MTTD** (Mean Time To Detect) — how long before a breach is noticed.
  - **MTTR** (Mean Time To Respond) — how long to contain it.
  - Industry average MTTD is often **months** — most breaches are discovered very late.

---

## 1.4 Interplay Between Privacy & Security

### Privacy Without Security

> *"I don't collect any data, so I can't have a breach."*

This is largely a **fallacy** in practice:

- **Impracticality with modern web services:** You cannot build a functional banking app, email service, or e-commerce platform without storing user data. The moment you store data, you need security.
- **Involuntary Data Leakage:** Even if *you* collect minimal data, third-party libraries, analytics tools, CDN providers, and browser extensions may collect data from your users without your full awareness.

**Real-world examples:**

- **Truecaller:**
  - Users who installed the app unwittingly **uploaded their entire contact lists** (including non-users' names and numbers) to Truecaller's servers.
  - Non-users **never consented** to being in this database, yet their personal information (name, phone number) was exposed.
  - Lesson: Your privacy policy means nothing if third-party integrations or app permissions enable involuntary data collection from non-consenting parties.

- **Cambridge Analytica (2018):**
  - A third-party Facebook quiz app collected data not just from the ~270,000 consenting users, but from **all of their Facebook friends** — approximately **87 million profiles** harvested in total.
  - This data was used to build psychographic profiles for **political micro-targeting** (Brexit referendum, 2016 US presidential election).
  - Exposed the extreme danger of **permissive third-party API access** on social platforms.
  - Facebook's API allowed apps to harvest friends' data without those friends' consent — a privacy design failure, not just a policy failure.

### Security Without Privacy

> *"We encrypt everything and pass all security audits."*

A system can be technically secure but still be a **massive privacy violation**:

- **High security + permissive data sharing:**
  - Data is protected from external attackers but is openly shared with or sold to advertisers, data brokers, or other businesses.
  - Example: A social media platform with excellent security that monetizes user behavioral data — the data is "secure" but users have **no meaningful privacy**.

- **Ad Networks & Profiling:**
  - Google's ad network tracks user behavior across millions of websites via embedded scripts and cookies.
  - Meta's tracking pixels are present on **healthcare, financial, and government websites** — recording sensitive behavior even on non-Meta properties.

- **Third-Party Data Brokering:**
  - Companies like Acxiom, Oracle Data Cloud, and Experian build profiles of individuals from thousands of data sources and sell them to marketers, insurers, employers, and governments.
  - This is largely **legal** and the data is often **technically secured** — but it represents a profound privacy violation.

---

## 1.5 Classification of Sensitive Information

### Direct PII
Data that **directly and unambiguously identifies an individual**:
- **Credentials:** Passwords, PINs, security answers, biometric templates.
- **Financial/Banking Details:** Card numbers, CVVs, bank account numbers, tax IDs.
- **Medical Records:** Diagnoses, prescriptions, lab results, insurance details, mental health records.
- **Government IDs:** SSN (US), Aadhaar (India), passport numbers, driver's license numbers.

> ⚠️ Exposure of Direct PII leads to immediate, concrete harm: identity theft, financial fraud, medical discrimination, targeted phishing.

### Indirect / Behavioral Data
Data that **does not directly identify** but can be used to infer sensitive attributes or, when combined, to re-identify:
- **Behavioral Tracking:** Pages visited, links clicked, time-on-page, scroll patterns, search queries.
- **Purchase History:** What you buy, how often, where — reveals lifestyle, health, religion, political views.
- **Recommendation Profiling:** Interests, vulnerabilities, emotional state, political leaning inferred from engagement patterns.
- **Location Data:** Home vs. work location, places of worship, medical facilities visited — extremely sensitive.

> 📌 **The aggregation problem:** None of these data points alone seems sensitive, but aggregated over time they form a detailed portrait of a person's life.

### Metadata
Data **about** communications or actions — not the content itself:
- **Communication Metadata:** Who you called/emailed, when, how long, how often — not the content.
- **Session Context:** IP address, device type, browser, login time, session duration.
- **Access Timestamps:** When files were accessed, modified, or deleted.

> 💬 *"We kill people based on metadata."* — Former NSA/CIA Director Michael Hayden (2014).

- Metadata reveals **relationships, habits, locations, and activities** — intelligence agencies find it as valuable as (or more than) content itself.

---

## 1.6 Regulatory Frameworks

### GDPR — General Data Protection Regulation (EU, 2018)
- **Jurisdiction:** Applies to *any* organization processing data of EU residents, regardless of where the organization is based (**extraterritorial reach**).
- **Key Principles:**
  1. **Lawfulness, Fairness, and Transparency** — Must have a legal basis for processing.
  2. **Purpose Limitation** — Data collected for one purpose cannot be used for another.
  3. **Data Minimization** — Only collect what is necessary.
  4. **Accuracy** — Keep data accurate and up to date.
  5. **Storage Limitation** — Don't keep data longer than needed.
  6. **Integrity and Confidentiality** — Appropriate security measures.
  7. **Accountability** — Must demonstrate compliance.
- **Fines:** Up to **€20 million** or **4% of global annual turnover**, whichever is higher.
  - Meta: €1.2 billion (2023) | Amazon: €746 million (2021) | Google: €50 million (2019)
- **72-hour breach notification** to the supervisory authority is mandatory.

### HIPAA — Health Insurance Portability and Accountability Act (US, 1996)
- **Jurisdiction:** US healthcare sector — covers "covered entities" and their "business associates."
- **Protects:** **PHI (Protected Health Information)** — any health information linked to an individual.
- **Key Rules:**
  - **Privacy Rule:** Limits how PHI can be used/disclosed; gives patients rights over their data.
  - **Security Rule:** Requires administrative, physical, and technical safeguards for electronic PHI (ePHI).
  - **Breach Notification Rule:** Notify affected individuals (and HHS) within **60 days** of discovering a breach.
- **Fines:** $100–$50,000 per violation, capped at $1.9M per violation category per year.

### Other Key Regulations

| Regulation | Region | Domain |
|---|---|---|
| **CCPA / CPRA** | California, USA | Consumer data privacy (broad) |
| **DPDPA** | India | Digital Personal Data Protection Act (2023) |
| **PIPEDA** | Canada | Private-sector organizations |
| **LGPD** | Brazil | General data protection |
| **COPPA** | USA | Children's online privacy (under 13) |
| **PCI-DSS** | International | Payment card data security |

### Legal Liability & Developer Obligations
- **Organizations:** Fines, lawsuits, mandatory audits, forced service shutdowns.
- **Individuals:** Developers and DPOs can face **personal liability** in some jurisdictions.
- **Developer obligations under GDPR:**
  - Implement privacy and security controls **by default**.
  - Maintain records of processing activities (Article 30).
  - Report breaches to supervisory authority within **72 hours**.
  - Appoint a **Data Protection Officer (DPO)** if processing at scale.
  - Conduct **Data Protection Impact Assessments (DPIAs)** for high-risk processing.

---

## 1.7 High-Level Mitigation Strategies

### Principle of Data Minimization
> *"The best data to protect is data you don't have."*

- **Definition:** Only collect data that is strictly necessary for the stated purpose. Do not collect "just in case."
- **Privacy-by-design:** Build systems from the ground up to collect the minimum viable data.

#### Case Study: Signal vs WhatsApp

| Aspect | Signal | WhatsApp |
|---|---|---|
| **Owner** | Signal Foundation (non-profit) | Meta (Facebook) |
| **Data collected** | Phone number only | Phone number, contacts, usage patterns, device info, IP addresses, location |
| **Message metadata** | Minimal — sealed sender protocol | Stored (who messages whom, frequency, timing) |
| **Backup encryption** | End-to-end encrypted backups | Historically unencrypted cloud backups |
| **Business model** | Donations | Advertising (data-driven) |
| **Regulatory footprint** | Minimal data = minimal surface area | Extensive data = extensive obligations |

> **Lesson:** The difference is not just ethics — it's *architecture*. Signal's design makes mass surveillance technically impossible even if legally compelled. WhatsApp's design makes it trivially easy.

### Split Responsibility: Frontend vs Backend

| Layer | Responsibility |
|---|---|
| **Frontend (Browser)** | Input validation (UX-level), CSP headers, secure cookie flags, HTTPS enforcement, preventing DOM-based XSS |
| **Backend (Server)** | Authoritative validation & sanitization, authentication & authorization, encryption at rest, secure credential storage, audit logging, access control enforcement |

> ⚠️ **Critical Rule:** **Never trust the frontend for security.** Frontend validation is for *user experience*; backend validation is for *security*. All security-critical checks must be enforced server-side.

---

## Summary: Key Takeaways

| Concept | One-Liner |
|---|---|
| **Privacy** | Control over who accesses your data |
| **Security** | Mechanisms to prevent unauthorized access |
| **Privacy ≠ Security** | You can have one without the other |
| **PII** | Any data that identifies or can re-identify a person |
| **Metadata** | Can be as sensitive as content itself |
| **GDPR** | Extraterritorial; fines up to 4% of global revenue |
| **HIPAA** | US healthcare; 60-day breach notification |
| **Data Minimization** | Don't collect what you don't need |
| **Frontend/Backend split** | Frontend = UX; Backend = security enforcement |

---

*Next → Section 2: Frontend Security*

## Further practice

- Revisit the examples in this topic and explain each step in your own words.
- Identify one place where the concept could be used in a web application.
- Test a small variation and note how the behavior changes.

