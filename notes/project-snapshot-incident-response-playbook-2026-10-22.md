## 🗂️ Project Snapshot — Incident Response Playbook (Home Lab)

> Deskripsi project lengkap — siap dipakai sebagai context di LinkedIn, CV, resume, atau web portofolio.

**Nama Project:** Incident Response Playbook — Home Lab
**Periode:** Oktober 2026 — masih berjalan
**Kategori:** Blue Team / SOC Analyst / SIEM & Network Security Monitoring
**Stack/Tools:** Wazuh (SIEM/EDR), Suricata (Network IDS), VirtualBox, Kali Linux, DVWA, Metasploitable2, Hydra, Nmap, iptables, Bash/jq scripting

**Deskripsi:**
Home lab berbasis VirtualBox yang mensimulasikan workflow Incident Response end-to-end (Detection → Analysis → Containment → Eradication → Recovery) terhadap beberapa skenario serangan nyata. Project ini menggabungkan deteksi host-based (Wazuh) dan network-based (Suricata) terhadap serangan SSH brute force dan web application attack (SQL Injection, webshell upload/RCE), termasuk investigasi mendalam terhadap infrastruktur monitoring yang gagal berfungsi dan perbaikannya. Setiap skenario didokumentasikan sebagai incident report profesional berbahasa Inggris, dilengkapi playbook terstruktur (YAML) dan bukti screenshot lengkap.

**Yang sudah dikerjakan:**
- Skenario 1 (SSH Brute Force) selesai penuh: recon, eksekusi, deteksi, root cause troubleshooting 3-lapis (service manager mati, service lain ikut mati, agent ID mismatch), containment, audit, writeup lengkap
- Skenario 2 (Web Application Attack) selesai penuh: deploy & troubleshoot Suricata dari nol, integrasi ke Wazuh, SQL Injection (2 varian), webshell upload & RCE, root cause 2 detection gap (config EXTERNAL_NET, missing signature), custom rule, containment & eradication, writeup lengkap
- SOP startup & troubleshooting lab terdokumentasi berdasarkan pengalaman nyata (bukan teori)
- README project, playbook YAML, dan writeup markdown terstruktur rapi per skenario

**Angka & bukti konkret:**
- 2 skenario serangan selesai end-to-end (dari 3 direncanakan)
- 2 playbook YAML terstruktur
- 9 file writeup markdown (incident report format)
- 5+ root cause investigation terdokumentasi lengkap (manager failed, indexer killed, agent ID mismatch, EXTERNAL_NET misconfiguration, rule path salah)
- 1 custom Suricata detection rule dibuat dari nol
- 90+ screenshot bukti terorganisir per skenario dan fase
- 2 VM baru dibangun & dikonfigurasi (Suricata IDS, dan rebuild Metasploitable2)

**5 Screenshot Terpenting:**

| No | Nama File (dengan path folder) | Apa yang ditunjukkan |
|---|---|---|
| 1 | `docs/scenario-01-bruteforce/response/08-root-cause-agent-id-mismatch.png` | Root cause kunci skenario 1 — agent ID tidak cocok akibat perubahan hostname, menyebabkan active response gagal |
| 2 | `docs/scenario-01-bruteforce/response/11-iptables-rule-drop-confirmed.png` | Bukti containment berhasil — IP attacker ter-block di firewall setelah fix |
| 3 | `docs/scenario-02-webapp/attack/20-webshell-rce-confirmed-uid-output.png` | Remote Code Execution berhasil dicapai via webshell upload (uid=33 www-data) |
| 4 | `docs/scenario-02-webapp/response/05-cek-struktur-folder-rules-lengkap.png` | Root cause custom rule gagal load — kesalahan lokasi file relatif terhadap default-rule-path |
| 5 | `docs/scenario-02-webapp/response/07-custom-rule-webshell-berhasil-detect.png` | Custom Suricata rule berhasil mendeteksi akses webshell setelah fix |

---

## 📋 Project Progress Checklist — Incident Response Playbook (Home Lab)

**Terakhir diupdate:** 22 Oktober 2026

### Skenario 1 — SSH Brute Force

| Status | Item | Catatan |
|---|---|---|
| ✅ | Setup lab awal (Kali, Wazuh Server, Endpoint) | |
| ✅ | Reconnaissance (Nmap) | Port 22 ditemukan |
| ✅ | Wordlist preparation & eksekusi brute force (Hydra) | Login berhasil |
| ✅ | Detection & analysis | Rule 40112 & 100002 |
| ✅ | Root cause troubleshooting (3 lapis) | Manager, indexer, agent ID |
| ✅ | Containment (active response + manual) | Session kill, password reset |
| ✅ | Audit bash history | |
| ✅ | Playbook YAML | `playbook-bruteforce.yml` |
| ✅ | Writeup lengkap (5 file .md) | |

### Infrastruktur Pendukung

| Status | Item | Catatan |
|---|---|---|
| ✅ | README project root | |
| ✅ | SOP Startup & Troubleshooting | Di `notes/` |
| ✅ | Struktur folder final disepakati | writeup per-skenario |

### Skenario 2 — Web Application Attack

| Status | Item | Catatan |
|---|---|---|
| ✅ | Deploy & troubleshoot VM Suricata | Rules, service, promiscuous mode |
| ✅ | Integrasi Suricata ke Wazuh | Agent ID 003 |
| ✅ | Reconnaissance (Nmap full scan) | 20+ port |
| ✅ | SQL Injection (error-based & bypass) | |
| ✅ | Webshell upload & RCE | uid=33 www-data |
| ✅ | Root cause detection gap (EXTERNAL_NET) | Fixed |
| ✅ | Custom Suricata rule (webshell) | SID 9000001 |
| ✅ | Containment & eradication | Urutan terbalik, dicatat sbg lesson |
| ✅ | Playbook YAML | `playbook-webapp.yml` |
| ✅ | Writeup lengkap (4 file .md) | |
| ⬜ | Recovery phase lengkap | Masih "pending" di playbook |

### Skenario 3 — Simulated Ransomware/Malware

| Status | Item | Catatan |
|---|---|---|
| ⬜ | Seluruh skenario | Baru rencana awal, belum dieksekusi |

### Lain-lain

| Status | Item | Catatan |
|---|---|---|
| ⬜ | Union-based SQLi | Diputuskan untuk di-skip, tidak ada di roadmap resmi |
| ⬜ | Cleanup repo final (folder duplikat, rename) | Beberapa instruksi sudah diberikan, belum dikonfirmasi selesai dieksekusi user |

---

### 📊 Summary

| Metric | Jumlah |
|---|---|
| ✅ Selesai | 29 item |
| 🔄 Sedang berjalan | 0 item |
| ⬜ Belum dimulai | 4 item |
| **Total** | **33 item** |
| **Progress** | **~88%** |

**Terakhir selesai di:** Writeup lengkap skenario 2 (README + 3 file detail) dan playbook-webapp.yml selesai dibuat.
**Selanjutnya:** Mulai Skenario 3 (Simulated Ransomware/Malware), atau selesaikan Recovery phase skenario 2 dan cleanup struktur repo final.
