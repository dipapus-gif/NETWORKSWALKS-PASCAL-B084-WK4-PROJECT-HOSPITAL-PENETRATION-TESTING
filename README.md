# NETWORKSWALKS-PASCAL-B084-WK4-PROJECT-HOSPITAL-PENETRATION-TESTING

## Mediroza General Hospital — Web Application Security Assessment

**Black-box Penetration Test**  
**Target:** https://medirozahospital.com  
**Duration:** 5 Days  
**Report Date:** 23 November 2026  
**Overall Risk:** CRITICAL  

This repository contains the deliverables for the Networkwalks Week 4 Penetration Testing Project.

### Contents

- `wk 4/Mediroza_Pentest_Report_Full.pdf` — Full professional penetration testing report
- `wk 4/Mediroza_Pentest_Report.pdf` — Report (from project ZIP)
- `wk 4/Networkwalks_Week4_Project_v1.pdf` — Project brief / milestones
- `wk 4/patient_report_*.pdf` — Decrypted patient lab reports (PHI recovered)
- `wk 4/RECON*.png` and screenshots — Evidence of exploitation
- Other supporting images from the assessment

### Key Findings Summary

| ID | Finding | Severity |
|----|---------|----------|
| F-01 | Publicly exposed HR database backup | CRITICAL |
| F-02 | SQL injection authentication bypass | CRITICAL |
| F-03 | Broken access control / IDOR on report download | HIGH |
| F-04 | Weak PDF encryption & weak passwords on lab reports | HIGH |
| F-05 | Sensitive path disclosure via robots.txt | MEDIUM |
| F-06 | Server & CMS version information disclosure | LOW |

**Milestones completed:**
- **M1:** Initial access via SQLi + IDOR → 3 patient PDF reports retrieved
- **M2:** Offline cracking of all 3 encrypted reports (rockyou)
- **M3:** Exposed `/old/mediroza_db_backup_2019.sql` containing staff salaries & shareholder data
- **M4:** Professional report

> This assessment was conducted in a controlled environment for educational purposes under the Networkwalks Week 4 programme. The target was explicitly authorised for security testing. These techniques must never be applied to any system without explicit written permission from its owner.
