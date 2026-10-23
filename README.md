# Incident Response Playbook — Home Lab

A self-built home lab project simulating end-to-end incident response workflows using Wazuh SIEM/EDR and Suricata IDS.

---

## Deskripsi

Project ini mensimulasikan siklus penuh Incident Response (Detection → Analysis → Containment → Eradication → Recovery) terhadap beberapa skenario serangan nyata, menggabungkan deteksi host-based (Wazuh) dan network-based (Suricata). Setiap skenario didokumentasikan sebagai incident report profesional, dilengkapi playbook terstruktur dan bukti lengkap, sebagai portofolio hands-on untuk peran SOC Analyst / Blue Team.

## Tech Stack & Tools

- **Wazuh** — SIEM/EDR untuk host-based log analysis, correlation rules, dan active response
- **Suricata** — Network IDS untuk deteksi berbasis traffic, termasuk custom signature
- **VirtualBox** — Virtualisasi seluruh lab (attacker, victim, monitoring)
- **Kali Linux** — Platform serangan (Nmap, Hydra)
- **DVWA & Metasploitable2** — Target aplikasi dan sistem yang sengaja rentan
- **Bash, jq, iptables** — Investigasi, analisis log, dan containment manual

## Struktur Folder

```
incident-response-playbook/
│
├── README.md
│
├── docs/
│   ├── setup/
│   ├── scenario-01-bruteforce/
│   │   ├── attack/
│   │   ├── detection/
│   │   └── response/
│   └── scenario-02-webapp/
│       ├── setup/
│       ├── attack/
│       ├── detection/
│       └── response/
│
├── playbooks/
│   ├── playbook-bruteforce.yml
│   └── playbook-webapp.yml
│
├── notes/
│   ├── SOP-startup-troubleshooting.md
│   └── troubleshooting.md
│
└── writeup/
    ├── README.md
    ├── scenario-01/
    │   ├── README.md
    │   ├── 01-reconnaissance-and-setup.md
    │   ├── 02-exploitation.md
    │   ├── 03-detection-and-analysis.md
    │   └── 04-root-cause-and-remediation.md
    └── scenario-02/
        ├── README.md
        ├── 01-infrastructure-and-reconnaissance.md
        ├── 02-exploitation.md
        └── 03-detection-and-remediation.md
```

## Progress

| Fase | Status |
|---|---|
| Skenario 1 — SSH Brute Force | ✅ Selesai |
| Skenario 2 — Web Application Attack (SQLi + Webshell) | ✅ Selesai |
| Skenario 3 — Simulated Ransomware/Malware | ⬜ Belum dimulai |

**Progress keseluruhan: ~88%**

## Dokumentasi

Writeup lengkap tersedia di folder `writeup/` — lihat [`writeup/README.md`](writeup/README.md) untuk navigasi per skenario. SOP operasional dan panduan troubleshooting lab tersedia di [`notes/SOP-startup-troubleshooting.md`](notes/SOP-startup-troubleshooting.md).

## Disclaimer

Project ini dibuat untuk tujuan pembelajaran dan portofolio pribadi, dijalankan sepenuhnya dalam lingkungan lab terisolasi (VirtualBox, jaringan Host-Only) menggunakan sistem dan aplikasi yang memang dirancang untuk latihan keamanan (DVWA, Metasploitable2). Teknik dan tools yang didemonstrasikan di sini tidak ditujukan untuk digunakan terhadap sistem apa pun tanpa izin eksplisit.
