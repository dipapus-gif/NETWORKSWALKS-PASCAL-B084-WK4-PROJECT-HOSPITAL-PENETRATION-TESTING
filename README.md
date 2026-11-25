# Penetration Testing Report
## Mediroza General Hospital — Web Application Security Assessment

![Networkwalks](https://img.shields.io/badge/Networkwalks-Week%204-0A66C2?style=for-the-badge) 
![Overall Risk](https://img.shields.io/badge/Overall%20Risk-CRITICAL-FF0000?style=for-the-badge) 
![Type](https://img.shields.io/badge/Engagement-Black--box-orange?style=for-the-badge) 
![Platform](https://img.shields.io/badge/Platform-Kali%20Linux-black?style=for-the-badge)

---

### Project Information

| Field                    | Details                                      |
|--------------------------|----------------------------------------------|
| **Client**               | Mediroza General Hospital                    |
| **Target**               | https://medirozahospital.com                 |
| **Engagement Type**      | Black-box Penetration Test                   |
| **Duration**             | 5 Days                                       |
| **Report Date**          | 23 November 2026                             |
| **Overall Risk Rating**  | **CRITICAL**                                 |
| **Classification**       | Confidential — Authorised Personnel Only     |
| **Pen-Tester**           | **Pascal Anorue** (Batch B084)               |
| **Instructor / Mentor**  | **Waqas Karim, CCIE** — Networkwalks         |
| **Platform**             | Kali Linux                                   |
| **Authorisation**        | Written permission granted                   |

> This assessment was conducted in a controlled environment for educational purposes under the **Networkwalks Week 4** programme, under the guidance of **Waqas Karim, CCIE**. The target was explicitly authorised for security testing. These techniques must never be applied to any system without explicit written permission from its owner.

---

## 01 Executive Summary

A comprehensive black-box penetration test was performed against the Mediroza General Hospital public web application. The engagement successfully identified a critical chain of vulnerabilities that, when combined, allow a completely unauthenticated remote attacker to:

- Bypass authentication on the patient portal using SQL Injection
- Enumerate and download every patient’s confidential laboratory reports via Insecure Direct Object Reference (IDOR)
- Crack the weak encryption protecting those reports offline in seconds
- Discover and download a publicly accessible internal database backup containing the salaries of all 30 employees and the full shareholder register

No privileged access, source code, or insider knowledge was required. Every vulnerability was reachable from the public internet.

### Key Metrics

| Metric                        | Result     |
|-------------------------------|------------|
| Total Findings                | 6          |
| Critical Findings             | 2          |
| High Findings                 | 2          |
| Medium Findings               | 1          |
| Low Findings                  | 1          |
| Patient Lab Reports Recovered | **3 / 3**  |
| Employee Salaries Exposed     | **30**     |
| Shareholders Exposed          | **10**     |

**Business Impact**  
This exposure represents a severe breach of patient confidentiality (Protected Health Information) as well as employee and corporate financial data. Under South Africa’s Protection of Personal Information Act (POPIA), such unauthorised disclosure constitutes a reportable data breach with significant regulatory, legal, financial and reputational consequences. Immediate remediation is required.

---

## 02 Scope and Methodology

### 2.1 Scope

| Category              | Details                                                                 |
|-----------------------|-------------------------------------------------------------------------|
| **In Scope**          | https://medirozahospital.com and all sub-paths                          |
| **Out of Scope**      | Social engineering, denial-of-service testing, any host outside the target domain |
| **Approach**          | Full black-box — no credentials, source code, or architecture documentation provided |
| **Rules of Engagement** | Target domain only · No social engineering · No denial of service · No testing outside agreed scope |
| **Authorisation**     | Written permission granted by the client (Networkwalks laboratory engagement) |

### 2.2 Methodology

The engagement followed a structured, industry-aligned methodology based on the **OWASP Testing Guide** and standard penetration testing lifecycle:

| Phase | Activities Performed |
|-------|----------------------|
| **1. Reconnaissance** | Passive and active footprinting, review of `robots.txt` and `sitemap.xml`, directory and file brute-forcing, technology fingerprinting |
| **2. Authentication Testing** | Analysis of login form behaviour, error message verbosity, session handling, and credential validation logic |
| **3. Injection & Input Validation** | Systematic testing for SQL Injection on all user-controlled parameters |
| **4. Access Control Testing** | Horizontal and vertical privilege escalation attempts, IDOR testing on object references |
| **5. Post-Exploitation & Data Analysis** | Offline cryptographic analysis of recovered files, examination of exposed sensitive data |

The work was organised into the four official Networkwalks milestones:

| Milestone | Title                  | Objective |
|-----------|------------------------|-----------|
| **M1**    | Initial Access         | Attack the website and retrieve the 3 confidential patient PDF laboratory reports |
| **M2**    | Data Extraction        | Crack the encryption on all 3 retrieved files |
| **M3**    | Critical Data Exposure | Locate staff salaries and shareholder details of the hospital |
| **M4**    | Professional Reporting | Produce a complete, client-ready penetration testing report |

### 2.3 Tools & Technologies Used

| Category                  | Tool / Technology                          | Purpose |
|---------------------------|--------------------------------------------|---------|
| **Operating System**      | Kali Linux                                 | Primary offensive security platform |
| **HTTP Client**           | curl                                       | Manual request crafting, header inspection, cookie and session handling |
| **Content Discovery**     | robots.txt, sitemap.xml, directory listing | Identification of hidden and sensitive paths |
| **PDF Analysis**          | qpdf                                       | Encryption scheme analysis (Standard Security Handler) and decryption |
| **Password Recovery**     | pdf2john + John the Ripper                 | Offline cracking of PDF passwords using the rockyou wordlist |
| **Document Extraction**   | poppler-utils (pdftotext, pdfinfo)         | Extraction of plaintext content from recovered laboratory reports |
| **Browser Tools**         | Browser Developer Tools                    | Live traffic inspection and response analysis |

---

## 03 Findings and Proof of Exploitation

### F-01 — Publicly Exposed HR Database Backup  
**Severity: CRITICAL** | Milestone: M3 | CWE-538 / CWE-200

**Location:** `/robots.txt` → `/old/` → `/old/mediroza_db_backup_2019.sql`

The application’s `robots.txt` file explicitly discloses a non-indexed directory (`/old/`). Directory listing is enabled on this path, allowing any anonymous internet user to download a complete SQL database backup without authentication.

The backup contains two highly sensitive tables:
- `staff` — full salary details, national identity numbers and contact information for all **30 employees**
- `shareholders` — ownership percentages and share holdings for all **10 shareholders**

This finding alone constitutes a critical data breach and was the highest-impact issue discovered during the engagement.

---

### F-02 — SQL Injection Authentication Bypass  
**Severity: CRITICAL** | Milestone: M1 | CWE-89 / CWE-287

**Location:** `POST /patient/login.php` (username parameter)

The patient portal login functionality fails to properly parameterise user input. By submitting the classic SQL comment payload:

```http
username=admin' --
```

the password verification clause is neutralised. The application issues a valid session cookie and redirects the attacker to `portal.php` (HTTP 302), granting full authenticated access to the patient portal without any valid credentials.

---

### F-03 — Broken Access Control / Insecure Direct Object Reference (IDOR)  
**Severity: HIGH** | Milestone: M1 | CWE-639

**Location:** `GET /patient/download.php?id=1..3`

Patient laboratory reports are referenced using sequential, predictable numeric identifiers. The download endpoint performs **no server-side ownership validation**. Once an authenticated session is obtained (via F-02), an attacker can simply iterate the `id` parameter and retrieve every patient’s confidential report. Directory listing is also enabled on the `/patient/` path, further exposing the application structure.

---

### F-04 — Weak PDF Encryption and Weak Passwords  
**Severity: HIGH** | Milestone: M2 | CWE-326 / CWE-521

All three recovered laboratory reports were protected using the outdated PDF Standard Security Handler (Revision 3, 128-bit RC4 — a cipher class deprecated since 2015). The user passwords were extremely weak and were recovered offline in seconds using `pdf2john` and John the Ripper against the rockyou wordlist:

| # | Patient Name       | Lab Reference   | Password Recovered | Method   |
|---|--------------------|-----------------|--------------------|----------|
| 1 | Sipho Dlamini      | LR-2024-1187    | `123456`           | rockyou  |
| 2 | Priya Reddy        | LR-2024-1192    | `password`         | rockyou  |
| 3 | Emily Thompson     | LR-2024-1205    | `!@#$%^&`          | rockyou  |

File-level encryption provided no meaningful protection of the Protected Health Information contained within the documents.

---

### F-05 — Sensitive Path Disclosure via robots.txt  
**Severity: MEDIUM** | Milestone: M1 / M3 | CWE-200

The `robots.txt` file explicitly lists sensitive application paths (`/patient/`, `/staff/`, `/old/`). While `robots.txt` is intended only as a crawler directive, it effectively provided a roadmap that led directly to the critical database backup (F-01).

---

### F-06 — Server and CMS Version Information Disclosure  
**Severity: LOW** | Milestone: M1 | CWE-200

HTTP response headers and HTML meta tags disclose the web server technology (**LiteSpeed**) and the exact application version (**Mediroza CMS 1.4.2**). Combined with site-wide directory listing (autoindex), this information assists an attacker in identifying potentially known vulnerabilities.

---

## 04 Risk Rating Summary

| ID   | Vulnerability                                      | Severity   |
|------|----------------------------------------------------|------------|
| F-01 | Publicly exposed HR database backup                | **CRITICAL** |
| F-02 | SQL Injection authentication bypass                | **CRITICAL** |
| F-03 | Broken Access Control / IDOR on report download    | **HIGH**     |
| F-04 | Weak PDF encryption & weak passwords               | **HIGH**     |
| F-05 | Sensitive path disclosure via robots.txt           | **MEDIUM**   |
| F-06 | Server & CMS version information disclosure        | **LOW**      |

---

## 05 Recommendations and Remediation

### Immediate (Critical)

1. **Remove the exposed database backup** (`mediroza_db_backup_2019.sql`) immediately and permanently disable directory listing on the `/old/` path and across the entire application.
2. **Remediate the SQL Injection vulnerability** by implementing parameterised queries (prepared statements) or a secure ORM. Never concatenate untrusted user input into SQL statements.
3. **Enforce strict access control** on the report download functionality. The application must verify, on the server side, that the authenticated user is authorised to access the specific report being requested.

### High Priority

4. Replace weak, password-based PDF encryption with strong, randomly generated passphrases of at least 16 characters, or preferably eliminate client-side password protection entirely in favour of authenticated portal-based distribution.
5. Serve all laboratory reports through an authenticated controller that streams files from a location outside the web root, rather than exposing direct file links.

### General Security Hardening

6. Implement robust session management (Secure and HttpOnly cookie flags, appropriate session lifetime, regeneration of session identifiers upon privilege change).
7. Enforce HTTPS site-wide with HTTP Strict Transport Security (HSTS) and automatic redirection of all HTTP requests.
8. Implement comprehensive security logging and real-time alerting for authentication failures, SQL errors, and access attempts to sensitive paths.
9. Disable directory listing (autoindex) across the entire web server configuration.
10. Establish a regular schedule of vulnerability assessments, secure code reviews, and penetration testing.

---

## 06 Evidence Collected

All supporting evidence for this engagement is stored in the `wk 4/` directory of this repository and includes:

- Full professional penetration testing report (`Mediroza_Pentest_Report_Full.pdf`)
- Three decrypted patient laboratory reports
- Extensive reconnaissance and exploitation screenshots
- Project brief and milestone documentation

---

## 07 Conclusion

The Mediroza General Hospital web application currently contains multiple trivially exploitable vulnerabilities that have already resulted in the unauthorised disclosure of confidential patient laboratory reports, the complete staff payroll, and the hospital’s shareholder register.

The two Critical findings (public database backup and SQL Injection authentication bypass) represent an immediate and severe risk to the organisation and must be remediated as a matter of the highest priority.

This engagement successfully completed all four official Networkwalks Week 4 milestones and delivered a complete, professional-grade penetration testing report.

---

### Credits & Acknowledgements

**Pen-Tester:**  
**Pascal Anorue**  
Batch B084 — Networkwalks Offensive Security Programme

**Instructor & Mentor:**  
**Waqas Karim, CCIE**  
Networkwalks

Special thanks to **Waqas Karim, CCIE** and the entire Networkwalks team for designing a realistic, challenging, and highly educational hands-on laboratory environment. The structured milestones and emphasis on both technical exploitation and professional reporting significantly accelerated my practical offensive security skills.

---

**Disclaimer**  
This project was conducted strictly for educational purposes inside an authorised laboratory environment provided by Networkwalks. The techniques demonstrated must never be used against any system without the explicit written permission of the system owner.

---

#PenetrationTesting #CyberSecurity #EthicalHacking #WebAppSec #Networkwalks #InfoSec #KaliLinux #OWASP #RedTeam #BugBounty #WaqasKarim #CCIE
