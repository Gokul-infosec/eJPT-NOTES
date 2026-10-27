# Auditing Fundamentals

> Status: ✅ Reference complete
> Date started: 2026-10-27
> Date finished: 2026-10-27

## 1. Goal
The concepts behind security auditing — what it is, how it differs from a pentest, the lifecycle, audit types, and compliance standards. Mostly theory (eJPT exam-relevant background), not commands.

## What is security auditing?
Systematic evaluation of an organization's security controls, processes and policies to check they're effective and compliant. Unlike a pentest (which exploits *technical* weaknesses), an audit starts at the **foundational** layers — org processes, policy adherence, operational procedures. Goals: find vulnerabilities/gaps, ensure compliance, manage risk, improve policies, support business objectives, enable continuous improvement.

## Audit vs. penetration test
| | Security Audit | Penetration Test |
|---|---|---|
| Purpose | evaluate posture + compliance with policy/standards | simulate real attacks, exploit vulns |
| Focus | controls, processes, practices | technical weaknesses, misconfigs |
| Scope | broad (policy, procedure, physical, technical) | defined systems/apps, agreed RoE |
| Method | doc review, interviews, technical checks, compliance verification | active exploitation, scanning, post-ex |
| Outcome | compliance status + policy/process gaps | exploited vulns + risk ratings + technical fixes |
| Frequency | regular (annual/biennial, or per compliance) | after major changes / scheduled |

**Sequential approach (common):** audit → remediation → pentest validates the fixes. Pentest scope is often driven by audit findings.

## Key terminology
- **Security policy** — formal doc defining security objectives/rules. **Compliance** — adherence to regs/standards/policy.
- **Vulnerability** — exploitable weakness. **Security control** — safeguard that prevents/detects/responds. **Risk assessment** — identify + analyse + prioritise risks (likelihood × impact).
- **Audit trail** — chronological record of events (accountability). **Access control** — who can do what. **Audit report** — formal findings + recommendations.

## Auditing lifecycle (7 phases)
1. Planning & preparation — objectives, scope, team, timeline, RoE, gather docs.
2. Information gathering — review policies, interview staff, collect configs, spot policy-vs-practice gaps.
3. Risk assessment — asset inventory, threat modelling, risk = likelihood × impact → prioritise.
4. Audit execution — technical testing, config reviews, compliance + control-effectiveness checks.
5. Analysis & evaluation — benchmark against standards (NIST, ISO 27001), prioritise by severity/impact.
6. Reporting — exec summary, findings, remediation guidance, stakeholder presentation.
7. Remediation & follow-up — fix, re-test, continuous monitoring → repeat the cycle.

## Audit types
| Type | What it covers | Pentest focus it informs |
|---|---|---|
| **Internal** | internal controls + policy compliance (own team) | policy validation, access control, social eng |
| **External** | independent third-party assessment / certification | perimeter testing, compliance verification |
| **Compliance** | regulatory adherence (GDPR/HIPAA/PCI DSS) | regulatory + data-protection testing |
| **Technical** | system/config hardening (CIS benchmarks) | infrastructure + config validation |
| **Network** | routers/switches/firewalls, segmentation, insecure protocols (Telnet, SNMP v1/2) | network exploitation, service enum |
| **Application** | code, authn/authz, input validation, OWASP Top 10 | web app + API testing |

## Compliance standards to know
- **GDPR** — EU personal-data protection. **HIPAA** — US healthcare data. **PCI DSS** — payment-card data. **ISO 27001** — ISMS (general infosec management). **SOC 2** — service-org controls. **NIST** / **CIS Benchmarks** — control + hardening frameworks.

> For the exam: know audit *vs* pentest, the lifecycle order, the audit types, and which standard maps to which data (health→HIPAA, cards→PCI DSS, EU personal data→GDPR). The hands-on part reuses the same tooling as the rest of the path (nmap, nikto, sslscan, sqlmap) — the difference is the goal (verify controls/compliance) not the tools.
