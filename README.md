# Cybersecurity Portfolio

> **Keelen Carrera** — CompTIA Security+ | Aspiring SOC Analyst / Security Analyst / Junior Penetration Tester

---

## Professional Summary

I am a cybersecurity practitioner pursuing entry-level roles in security operations and offensive security. This portfolio demonstrates hands-on skills across detection engineering, vulnerability management, and incident response — built entirely in home-lab environments using industry-standard tooling. Every project maps to real-world workflows that mirror what SOC teams, vulnerability management programs, and IR teams execute daily.

I hold the **CompTIA Security+** certification and am actively building depth in:
- SIEM engineering and threat detection (Wazuh, Splunk)
- Vulnerability scanning and remediation workflows (OpenVAS, Nessus Essentials)
- Incident response documentation and playbook development
- Adversary simulation via MITRE ATT&CK and Atomic Red Team

---

## Skills Matrix

| Skill | Project | NIST CSF Function | Security+ Domain |
|---|---|---|---|
| SIEM deployment & configuration | 01 — SOC Home Lab | Detect (DE) | 4.0 Security Operations |
| Custom Sigma detection rules | 01 — SOC Home Lab | Detect (DE) | 4.0 Security Operations |
| MITRE ATT&CK mapping | 01 — SOC Home Lab | Identify (ID) / Detect (DE) | 1.0 General Security Concepts |
| Adversary simulation (Atomic Red Team) | 01 — SOC Home Lab | Detect (DE) / Respond (RS) | 4.0 Security Operations |
| Endpoint & network hardening | 01 — SOC Home Lab | Protect (PR) | 3.0 Implementation |
| Vulnerability scanning & analysis | 02 — Vulnerability Management | Identify (ID) | 4.0 Security Operations |
| Risk scoring & prioritization (CVSS) | 02 — Vulnerability Management | Identify (ID) | 5.0 Security Program Management |
| Remediation tracking & reporting | 02 — Vulnerability Management | Respond (RS) / Recover (RC) | 5.0 Security Program Management |
| Incident response lifecycle | 03 — IR Playbook | Respond (RS) / Recover (RC) | 4.0 Security Operations |
| Playbook authoring | 03 — IR Playbook | Respond (RS) | 4.0 Security Operations |
| Evidence collection & chain of custody | 03 — IR Playbook | Respond (RS) | 4.0 Security Operations |

---

## Projects

### [01 — SOC Home Lab with SIEM Detection Engineering](./01-soc-home-lab/)

Build a fully functional home SOC using Wazuh (or Splunk Free) deployed across a three-VM lab. Includes:
- Full environment setup guide with network diagram
- 5 custom Sigma detection rules mapped to MITRE ATT&CK
- Detection walkthrough documents per rule
- Atomic Red Team simulation commands
- Hardening recommendations

**Tools:** VirtualBox · Wazuh · Kali Linux · Windows 10 VM · Ubuntu Server  
**ATT&CK Techniques:** T1110, T1059.001, T1003.001, T1021.002, T1053.005

---

### [02 — Vulnerability Management Program](./02-vulnerability-management/)

Simulate an enterprise vulnerability management lifecycle: scan, analyze, prioritize, remediate, and report. Includes:
- OpenVAS/Nessus scan setup and execution guide
- Annotated scan results with CVSS scoring analysis
- Risk-prioritized remediation tracker (CSV + markdown)
- Executive summary report template

**Tools:** OpenVAS · Nessus Essentials · Metasploitable 2  
**Frameworks:** CVSS v3.1 · NIST SP 800-40

---

### [03 — Incident Response Playbook](./03-incident-response-playbook/)

A documentation-heavy project producing analyst-ready IR playbooks for three common incident types. Includes:
- IR lifecycle overview (PICERL model)
- Playbooks for: ransomware, phishing, and unauthorized access
- Evidence collection checklists
- Post-incident report template

**Frameworks:** NIST SP 800-61r2 · PICERL · MITRE ATT&CK

---

## How to Navigate This Repo

```
cybersecurity-portfolio/
├── README.md                          ← You are here — portfolio overview
├── .gitignore
├── 01-soc-home-lab/
│   ├── README.md                      ← Lab setup guide + network diagram
│   ├── detections/                    ← Sigma rules (.yml)
│   │   └── walkthroughs/              ← Per-rule detection walkthroughs
│   ├── simulations/                   ← Atomic Red Team test commands
│   └── hardening-recommendations.md
├── 02-vulnerability-management/
│   ├── README.md
│   ├── scan-results/
│   ├── remediation-tracker.md
│   └── executive-summary-template.md
└── 03-incident-response-playbook/
    ├── README.md
    ├── playbooks/
    ├── evidence-collection-checklist.md
    └── post-incident-report-template.md
```

Each sub-project's `README.md` is a self-contained guide. Read them top-to-bottom to follow the workflow. Screenshot placeholders are marked with `[SCREENSHOT]` tags — these are replaced with actual captures during lab execution.

---

## Certifications

| Certification | Issuer | Status |
|---|---|---|
| CompTIA Security+ | CompTIA | Active |
| Certified Cloud Practitioner | AWS | Active |

---

## Contact

- GitHub: [github.com/keelen-carrera](https://github.com/keelen-carrera)
- LinkedIn: *[Keelen Carrera LinkedIn](https://www.linkedin.com/in/keelencarrera/)*
- Email: *keelencarrera@proton.me*
