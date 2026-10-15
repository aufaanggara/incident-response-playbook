# Incident Report — SSH Brute Force Attack (Home Lab)

**Date:** 2026-10-14 to 2026-10-15
**Platform:** Self-built Home Lab (VirtualBox + Wazuh SIEM/EDR)
**Difficulty:** Medium
**Category:** Intrusion / Brute Force / Active Response Troubleshooting
**Analyst:** Project owner (self-directed)

---

## Navigation

| # | Document | Content |
|---|----------|---------|
| 1 | [01-reconnaissance-and-setup.md](./01-reconnaissance-and-setup.md) | Lab environment, topology, reconnaissance (Nmap) |
| 2 | [02-exploitation.md](./02-exploitation.md) | Wordlist strategy, SSH brute force execution (Hydra) |
| 3 | [03-detection-and-analysis.md](./03-detection-and-analysis.md) | Detection rules, alert analysis, before/after comparison |
| 4 | [04-root-cause-and-remediation.md](./04-root-cause-and-remediation.md) | Active response troubleshooting, containment, audit |

---

## Executive Summary

A simulated SSH brute force attack was conducted against a Linux endpoint monitored by a self-hosted Wazuh SIEM in a controlled home lab environment. The attacker successfully authenticated to a valid account (`endpoint`) after a deliberately staged series of failed login attempts. Wazuh's correlation engine correctly detected the attack pattern (multiple failures followed by a success, and multiple non-existent-user failures), but the automated containment action (firewall block) initially failed to execute due to infrastructure issues stemming from the lab being idle for roughly two months. Root cause analysis identified three compounding issues: a stalled manager service, a collateral service shutdown, and a stale agent ID mapping caused by a hostname change. After remediation, the full detection-to-containment cycle was re-validated successfully: the attacker's IP was blocked at the firewall level, the compromised session was terminated, and the compromised account's password was reset.

---

## Indicators of Compromise (IOC)

| Type | Value | Notes |
|---|---|---|
| Source IP (Attacker) | `192.168.56.102` | Kali Linux VM used for scanning and brute force |
| Target IP (Victim) | `192.168.56.106` | Ubuntu 24.04.4 LTS endpoint, SSH exposed |
| Target Service | SSH (port 22), OpenSSH 9.6p1 Ubuntu 3ubuntu13.18 | Sole exposed service confirmed via Nmap |
| Compromised Account | `endpoint` | Valid local account on victim host |
| Attack Tool | `hydra v9.6` | Used for credential brute forcing |
| Malicious Usernames Attempted | `root`, `admin`, `user`, `test` | Non-existent accounts, triggered correlation rule 100002 |

> Note: the specific password value used in the wordlist and subsequently compromised is intentionally omitted from this report (credential hygiene), even though this is an isolated lab environment.

---

## Timeline of Events (High Level)

| Timestamp (local) | Event |
|---|---|
| 2026-07-27/28 | Initial Wazuh agent enrollment (previous project session), agent registered as `endpoint-linux-wazuh` (ID 001) |
| 2026-07-30 | Prior active-response testing performed successfully (historical baseline) |
| ~2026-07-30 to 2026-10-14 | Lab left idle (~2 months) |
| 2026-10-14 05:33 | Lab VMs restarted; agent re-enrolled under new hostname `endpoint-linux-waz-plunk` (ID 002) |
| 2026-10-14 ~10:37 | Nmap reconnaissance performed against victim |
| 2026-10-14 ~10:37 | Hydra brute force executed; valid login achieved on account `endpoint` |
| 2026-10-14 | Detection confirmed: rule 40112 (fail→success) and rule 100002 (non-existent user) both triggered; active response **did not execute** |
| 2026-10-15 05:33–06:19 | Root cause investigation: `wazuh-manager` found in failed state; `wazuh-indexer` found killed; both restarted |
| 2026-10-15 06:1x | Root cause identified: `agent_id` in active-response config (001) did not match actual registered agent ID (002) |
| 2026-10-15 06:19 | `ossec.conf` updated (`agent_id` 001 → 002); manager restarted |
| 2026-10-15 06:41 | Brute force re-executed; active response (`firewall-drop`) confirmed executing successfully |
| 2026-10-15 ~08:48–08:56 | Active attacker session on compromised account identified, terminated, password reset |

Full step-by-step detail with evidence is documented in the linked phase files above.

---

## Key Findings (Highlights)

- Reconnaissance confirmed a single attack surface: SSH (port 22).
- Wazuh's built-in correlation rule (ID 40112) detected the brute-force-to-success pattern even though the custom rule (ID 100002) was scoped only to non-existent-user failures — a useful detection coverage overlap.
- The custom rule (ID 100002) does **not** cover brute force against valid usernames — a detection gap identified during analysis.
- Automated containment (active response) silently failed on first attempt due to **three compounding infrastructure issues** after prolonged lab inactivity, not a configuration logic error in the rule itself.
- Root cause was ultimately traced to an **agent identity mismatch** caused by a hostname change, which desynced the active-response target (`agent_id`) from the actual registered agent.
- After remediation, full containment was verified at multiple levels: network (ping test), firewall (iptables rule), and application log (`active-responses.log`).
- Manual containment (session termination + password reset) was also demonstrated for the compromised account itself.

See [04-root-cause-and-remediation.md](./04-root-cause-and-remediation.md) for full root cause analysis, remediation steps, and lessons learned.
