# Ali Arsalan — Security Portfolio

**CISSP · OSCP · CCIE-Security · AWS Solutions Architect Professional / Security Specialty**
Cybersecurity engineer (15+ years: enterprise security architecture, cloud security, offensive testing). Current focus: **AI security — securing LLM-integrated applications and agentic systems**.

This repository hosts selected, **redacted** sample work. Full versions and additional case material available on request: **a.aly@live.com**

---

## Agentic Penetration Testing Platform

An end-to-end automated penetration-testing pipeline that combines traditional offensive-security tooling with LLM-driven analysis and multi-stage quality gates.

**Pipeline stages:**

1. **Recon & crawling** — target mapping, SPA-aware crawling, technology fingerprinting
2. **Vulnerability scanning** — Nmap, Metasploit, web-application scanners
3. **LLM analysis agent** — an AI agent reasons over scan output and raw traffic, plans exploitation steps, and drives interactive testing (SQLi, auth bypass, business-logic abuse)
4. **Automated QA gate** — a zero-context LLM judge re-verifies every claimed finding against the raw evidence; unsupported claims are rejected before they reach the report
5. **Report generation** — templated, commercial-grade reports (executive summary, technical findings with CVSS, remediation tracker) with automated **redaction and data-leak checks** before release

**Design principles:**

- Human-in-the-loop rules-of-engagement gating — no test runs without explicit authorization
- Evidence-first findings — every claim must trace to captured evidence (requests, responses, screenshots)
- Negative results recorded with equal structure to positives (tested-and-clean vs. not-tested vs. vulnerable)
- Two-person discipline built in: generator and judge are separate, independent passes

## Sample Reports (redacted)

**Sample reports (both redacted, both authorized):**

- **[Authorized assessment — outreachdesk.cloud](reports/outreachdesk-pentest-public.pdf)** — production SaaS (my own asset). Full pipeline output: auth testing, Cloudflare Turnstile bypass-resistance, API assessment.
- **[DVWA demonstration report](reports/dvwa-demo-report.pdf)** — methodology demonstration against a deliberately vulnerable public lab target.

Both reports are redacted and published with authorization — the production assessment targets an asset I own; the demonstration targets a public purpose-built lab.

## Architecture

![Agentic penetration-testing pipeline architecture](diagrams/pentest-framework-diagram.png)

Full diagram: [pentest-framework-diagram-public.pdf](diagrams/pentest-framework-diagram-public.pdf) — physical/logical topology (control plane, execution plane, verification plane) plus the end-to-end "step 1 → step N" workflow with toolchains, evidence flow, and QA gates.

---

## AI Security

Working knowledge and applied practice in: OWASP LLM Top 10, MITRE ATLAS, NIST AI RMF; prompt-injection assessment of LLM-integrated applications; RAG/vector-store attack surface; agent tool-permission and least-privilege design; open-weight model supply chain (Hugging Face, local inference runtimes); LLM-as-judge evaluation pipelines.

## Contact

- Email: **a.aly@live.com**
- LinkedIn: **[linkedin.com/in/ali-arsalan-2039976](https://www.linkedin.com/in/ali-arsalan-2039976/)**
