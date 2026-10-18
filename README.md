# Incident Response Playbook — Home Lab

A self-built home lab project simulating end-to-end incident response workflows (Detection → Analysis → Containment → Eradication → Recovery) using Wazuh SIEM/EDR and Suricata IDS, built and documented as a hands-on blue team portfolio.

---

## Project Structure

```
incident-response-playbook/
├── README.md                          ← you are here
├── SOP-startup-troubleshooting.md     ← operational checklist, read before starting any lab session
├── writeup/                           ← incident reports (per scenario, portfolio-ready)
│   ├── README.md                      ← Scenario 1 report index
│   ├── 01-reconnaissance-and-setup.md
│   ├── 02-exploitation.md
│   ├── 03-detection-and-analysis.md
│   └── 04-root-cause-and-remediation.md
├── playbooks/                         ← structured YAML playbooks (reusable SOP format)
│   └── playbook-bruteforce.yml
└── docs/                              ← raw evidence (screenshots), organized per scenario
    ├── setup/
    ├── scenario-01-bruteforce/
    └── scenario-02-webapp/
```

---

## Scenarios

| # | Scenario | Attack Vector | Status | Report |
|---|---|---|---|---|
| 1 | SSH Brute Force | Credential brute forcing (Hydra) against SSH | ✅ Complete | [writeup/README.md](./writeup/README.md) |
| 2 | Web Application Attack | SQL Injection / webshell upload (Metasploitable2 + Suricata) | 🔄 In progress | — |
| 3 | Simulated Ransomware/Malware | Safe sample execution, containment & recovery focus | ⏳ Planned | — |

---

## Lab Environment Overview

| Role | Notes |
|---|---|
| Attacker | Kali Linux |
| Victim(s) | Vary per scenario — Linux endpoint (Scenario 1), Metasploitable2 (Scenario 2) |
| Monitoring | Wazuh Manager/Indexer/Dashboard (SIEM/EDR), Suricata (network IDS, Scenario 2 onward) |
| Networking | VirtualBox Host-Only Adapter (inter-VM, isolated) + NAT (per-VM internet access) |

All VMs run on VirtualBox on a local Windows host. Full topology and per-scenario environment details are documented in each scenario's writeup.

---

## Before Starting Any Session

This lab is not kept running continuously — VMs are started and stopped between sessions, which has previously caused several infrastructure issues (stalled services, stale agent mappings, lost network configuration) that were mistaken for attack-related anomalies before their real cause was found.

**Always run through [SOP-startup-troubleshooting.md](./SOP-startup-troubleshooting.md) first** before beginning a new scenario or resuming an existing one. It covers:
- The correct VM startup order and post-boot verification checklist
- Step-by-step troubleshooting for the specific failure modes already encountered in this lab (Wazuh manager/indexer failures, active-response not firing, Suricata not capturing third-party traffic, stuck processes, and more)

Treating infrastructure health checks as a deliberate first step — rather than assuming the environment is ready — is itself part of the incident response discipline this project is meant to practice.

---

## Purpose

This project exists to practice and document the full workflow of a SOC/blue team analyst end to end: executing a realistic attack scenario, detecting and analyzing it through a SIEM, investigating when automated response doesn't behave as expected, performing containment and remediation, and writing it all up as a professional incident report. The written reports under `writeup/` are the primary portfolio deliverable; the YAML playbooks under `playbooks/` are a reusable, structured summary of each scenario's response procedure.
