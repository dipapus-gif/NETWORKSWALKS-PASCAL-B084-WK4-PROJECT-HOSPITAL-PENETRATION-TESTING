# Penetration Testing Report — Mediroza General Hospital

![Networkwalks](https://img.shields.io/badge/Networkwalks-Week%204-blue) ![Overall Risk](https://img.shields.io/badge/Overall%20Risk-CRITICAL-red) ![Type](https://img.shields.io/badge/Type-Black--box-important) ![Platform](https://img.shields.io/badge/Platform-Kali%20Linux-black)

**Client:** Mediroza General Hospital  
**Target:** https://medirozahospital.com  
**Engagement Type:** Black-box Penetration Test  
**Duration:** 5 Days  
**Report Date:** 23 November 2026  
**Overall Risk Rating:** **CRITICAL**  
**Authorisation:** Written permission granted (Networkwalks lab engagement)

> This assessment was conducted in a controlled environment for educational purposes under the Networkwalks Week 4 programme. The target was explicitly authorised for security testing. These techniques must never be applied to any system without explicit written permission from its owner.

---

## 01 Executive Summary

A black-box penetration test of the Mediroza General Hospital web application identified a chain of critical vulnerabilities that together allow an unauthenticated attacker to:

- Bypass patient portal authentication via SQL Injection
- Retrieve and decrypt all three protected patient laboratory reports
- Download a complete internal HR database backup containing staff salaries, national identity numbers and shareholder records

No special tooling, insider knowledge or privileged access was required — every issue is reachable from the public internet.

| Metric                    | Value      |
|---------------------------|------------|
| Total Findings            | 6          |
| Critical                  | 2          |
| High                      | 2          |
| Medium                    | 1          |
| Low                       | 1          |
| Patient Reports Recovered | 3 / 3      |
| Employee Salaries Exposed | 30         |
| Shareholders Exposed      | 10         |

**Key Business Impact**  
The exposure constitutes a severe breach of patient confidentiality (protected health information) and of employee & corporate financial data. Under South Africa’s POPIA, unauthorised disclosure of this nature is a reportable data breach carrying regulatory, legal and reputational consequences.

---

## 02 Scope & Methodology

### 2.1 Scope

| Item              | Detail                                              |
|-------------------|-----------------------------------------------------|
| In scope          | https://medirozahospital.com and all sub-paths      |
| Out of scope      | Social engineering, denial-of-service, any host outside the target domain |
| Approach          | Black-box — no credentials, source code or architecture documentation provided |
| Rules of engagement | Target domain only · No social engineering · No DoS · No testing outside agreed scope |
| Authorisation     | Written permission granted by the client (Networkwalks lab) |

### 2.2 Methodology

Testing followed a standard web-application assessment workflow aligned with the **OWASP Testing Guide**:

1. **Reconnaissance & Content Discovery** – robots.txt, sitemap.xml, directory listing enumeration
2. **Authentication Analysis** – login form behaviour, error messages, session handling
3. **Input Handling & Injection Testing** – SQL Injection on username parameter
4. **Access-Control Testing** – IDOR on report download endpoint
5. **Post-Exploitation Data Analysis** – offline PDF cracking + analysis of exposed database backup

Work proceeded through the four engagement milestones:

| Milestone | Objective |
|-----------|-----------|
| **M1**    | Initial Access – Retrieve 3 confidential patient PDF lab reports |
| **M2**    | Data Extraction – Crack the encryption on all 3 retrieved files |
| **M3**    | Critical Data Exposure – Find staff salaries and shareholder details |
| **M4**    | Professional Reporting – Write the full penetration testing report |

### 2.3 Tools Used

| Tool                        | Purpose |
|-----------------------------|---------|
| **Kali Linux**              | Primary testing platform |
| **curl**                    | Manual request crafting, header inspection, cookie-backed session handling |
| **robots.txt / sitemap.xml**| Content discovery and sensitive path identification |
| **qpdf**                    | PDF encryption analysis (Standard Security Handler) and decryption |
| **pdf2john + John the Ripper** | Offline PDF password recovery against the rockyou wordlist |
| **poppler-utils** (pdftotext, pdfinfo) | Content extraction from recovered reports |
| Browser DevTools            | Traffic inspection and response analysis |

---

## 03 Findings & Proof of Exploitation

### F-01 — Publicly Exposed HR Database Backup (CRITICAL)

- **Milestone:** M3  
- **CWE:** CWE-538 / CWE-200 – Exposure of Sensitive Information  
- **Location:** `/robots.txt` → `/old/` → `/old/mediroza_db_backup_2019.sql`

The site’s `robots.txt` discloses a hidden `/old/` directory. Directory listing is enabled, revealing a SQL database backup downloadable with **no authentication**. The backup contains two confidential tables:

- `staff` – salaries, national ID numbers and contact details for all **30 employees**
- `shareholders` – ownership percentages and share counts for all **10 shareholders**

This is the single most severe issue identified.

### F-02 — SQL Injection Authentication Bypass (CRITICAL)

- **Milestone:** M1  
- **CWE:** CWE-89 (SQL Injection) / CWE-287 (Improper Authentication)  
- **Location:** `POST /patient/login.php` (username parameter)

The patient portal login does not parameterise the username field. Supplying the classic payload:

```http
username=admin' --
```

causes the application to comment out the password check and issue a valid session (HTTP 302 → `portal.php`).

### F-03 — Broken Access Control / IDOR on Report Download (HIGH)

- **Milestone:** M1  
- **CWE:** CWE-639 – Insecure Direct Object Reference  
- **Location:** `GET /patient/download.php?id=1..3`

Reports are referenced by sequential, guessable numeric IDs. The endpoint performs **no ownership check**. Any authenticated session (obtained via F-02) can enumerate and download every patient’s report. Directory listing is also enabled on `/patient/`.

### F-04 — Weak PDF Encryption & Weak Passwords (HIGH)

- **Milestone:** M2  
- **CWE:** CWE-326 (Inadequate Encryption Strength) / CWE-521 (Weak Password Requirements)

All three lab reports use the legacy PDF Standard Security Handler (V2/R3, 128-bit RC4 — deprecated since 2015) protected by extremely weak user passwords. All three were recovered offline within seconds using `pdf2john` + John the Ripper against the rockyou wordlist:

| # | Patient          | Password Recovered | Method   |
|---|------------------|--------------------|----------|
| 1 | Sipho Dlamini    | `123456`           | rockyou  |
| 2 | Priya Reddy      | `password`         | rockyou  |
| 3 | Emily Thompson   | `!@#$%^&`          | rockyou  |

### F-05 — Sensitive Path Disclosure via robots.txt (MEDIUM)

`robots.txt` explicitly lists sensitive non-indexed paths (`/patient/`, `/staff/`, `/old/`), handing an attacker a map of exactly where to look. This was the pivot that led directly to the exposed database backup (F-01).

### F-06 — Server & CMS Version Information Disclosure (LOW)

HTTP response headers and HTML `<meta generator>` tags disclose the web server (**LiteSpeed**) and application version (**Mediroza CMS 1.4.2**). Directory listing (autoindex) is enabled site-wide.

---

## 04 Risk Rating

Each vulnerability is rated using a combination of technical exploitability and business impact to Mediroza General Hospital.

| ID   | Vulnerability                                      | Severity  |
|------|----------------------------------------------------|-----------|
| F-01 | Publicly exposed HR database backup                | CRITICAL  |
| F-02 | SQL Injection authentication bypass                | CRITICAL  |
| F-03 | Broken Access Control / IDOR on report download    | HIGH      |
| F-04 | Weak PDF encryption & weak passwords               | HIGH      |
| F-05 | Sensitive path disclosure via robots.txt           | MEDIUM    |
| F-06 | Server & CMS version information disclosure        | LOW       |

---

## 05 Recommendations & Remediation

### Critical (Immediate)

1. **Remove the exposed database backup** immediately and disable directory listing on `/old/` and site-wide.
2. **Fix the SQL Injection** – parameterise all queries (prepared statements / ORM). Never concatenate user input into SQL.
3. **Enforce proper access control** on `/patient/download.php` – verify the authenticated user’s ownership of the requested report server-side.

### High

4. Replace password-based PDF protection with strong, unique passphrases (16+ characters) **or preferably** a secure distribution mechanism (authenticated portal download only).
5. Serve reports through an authenticated controller that streams files from a location **outside the web root**.

### General Security Improvements

6. Implement secure session management (Secure + HttpOnly cookies, session expiry, regeneration on privilege change).
7. Enforce HTTPS everywhere with HSTS and redirect all HTTP traffic.
8. Add security logging and monitoring/alerting for authentication failures, SQL errors, and access to sensitive paths.
9. Disable directory listing across the entire site.
10. Conduct regular vulnerability assessments and code reviews.

---

## 06 Evidence Collected

All evidence is available in the `wk 4/` directory of this repository:

- Full professional report: `Mediroza_Pentest_Report_Full.pdf`
- Decrypted patient lab reports (3 files)
- Extensive reconnaissance and exploitation screenshots (RECON*.png + additional screenshots)
- Project brief and milestone document

---

## 07 Conclusion

Mediroza General Hospital’s web application is currently exposed to trivially exploitable flaws that have already disclosed confidential patient reports, the complete staff payroll, and the shareholder register. The Critical findings (F-01 and F-02) must be remediated as a matter of urgency.

This engagement successfully completed all four Networkwalks Week 4 milestones and produced a full professional penetration testing report.

---

**Pen-tester:** Pascal Chimdindu Anorue (Batch B084)  
**Instructor:** Networkwalks  
**Platform:** Kali Linux  
**Tools:** curl · qpdf · pdf2john · John the Ripper · poppler-utils

#PenetrationTesting #CyberSecurity #EthicalHacking #WebAppSec #Networkwalks #InfoSec #KaliLinux #OWASP
