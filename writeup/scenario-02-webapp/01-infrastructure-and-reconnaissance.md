[← Back to Scenario 2 Index](./README.md) | Next: [Exploitation →](./02-exploitation.md)

# Phase 1 — Infrastructure Setup & Reconnaissance

## 1.1 Why Network-Based Detection

Metasploitable2, by design, runs an obsolete OS (Ubuntu 8.04) incompatible with modern Wazuh agent packages. Rather than skip host-based monitoring entirely, a network-based IDS (Suricata) was deployed on a separate VM positioned to observe traffic between the attacker and the victim — a common real-world pattern for monitoring legacy or unmanageable assets.

## 1.2 Environment

| Role | Hostname | IP | Notes |
|---|---|---|---|
| Attacker | kali-linux | `192.168.56.102` | |
| Victim | metasploitable2 | `192.168.56.112` | Intentionally vulnerable; hosts DVWA |
| Network IDS | suricata-ids | `192.168.56.111` | New VM, Suricata v8.0.7 |
| SIEM | wazuh-server | `192.168.56.104` | Pre-existing from Scenario 1 |

## 1.3 Suricata Deployment Summary

Suricata was installed via the official OISF PPA (not the default Ubuntu repo, for a current release) and configured to monitor the Host-Only network interface (`enp0s8`). Several infrastructure issues were encountered and resolved during setup:

- **Rules not loaded on first run** — resolved by running `suricata-update` to fetch the Emerging Threats Open ruleset (~69,000 rules).
- **Suricata killed unintentionally when run in foreground** — running the engine manually and then typing further commands in the same terminal sent an interrupt signal, stopping it. Resolved by running Suricata as a `systemd` service instead.
- **Promiscuous mode not actually active** despite VirtualBox's "Allow All" setting — the hypervisor-level setting did not automatically propagate to the guest OS network stack. Verified with `ip link show` (no `PROMISC` flag present) and confirmed via `tcpdump`, which showed third-party traffic was visible only while `tcpdump` itself was running (it enables promiscuous mode as a side effect). Resolved by explicitly enabling it with `ip link set <iface> promisc on`, made persistent via a dedicated `systemd` oneshot service that runs before Suricata starts.

## 1.4 Wazuh Integration

The Suricata VM was enrolled as a new Wazuh agent (ID 003), and `ossec.conf` was extended with a `<localfile>` block pointing to `/var/log/suricata/eve.json` (format: `json`), allowing Suricata's alert and flow data to flow into the same Wazuh dashboard used in Scenario 1. An old, disconnected agent (ID 001, orphaned by a hostname change from the prior scenario) was removed for dashboard clarity.

## 1.5 Reconnaissance

**Full port scan:**
```bash
sudo nmap -sV -sC -p- -T4 -oN nmap-metasploitable2-fullscan.txt 192.168.56.112
```
A full port range (`-p-`) was used rather than the top-100 scan from Scenario 1, since Metasploitable2 intentionally exposes vulnerabilities across many non-standard ports.

**Result:** 20+ open ports, including FTP with anonymous login enabled, Telnet, an exposed "Metasploitable root shell" backdoor port (1524), and — the focus of this scenario — port 80 (Apache httpd 2.2.8, title "Metasploitable2 - Linux").

**Web enumeration:** accessing the root web page revealed links to five intentionally vulnerable applications: TWiki, phpMyAdmin, Mutillidae, DVWA, and WebDAV. DVWA was selected as the target, consistent with the original scenario plan (SQL Injection and file upload).

DVWA's security level was changed from its default ("high") to "low" via the application's built-in security settings, to establish a clear, unobstructed baseline for the first SQLi/upload attempts in this exercise.

---

[← Back to Scenario 2 Index](./README.md) | Next: [Exploitation →](./02-exploitation.md)
