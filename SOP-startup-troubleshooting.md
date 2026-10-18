# SOP — Lab Startup Checklist & Troubleshooting

Dipakai setiap kali lab (Wazuh Server, Suricata IDS, endpoint, Kali, Metasploitable2) dinyalakan ulang setelah idle, untuk memastikan semua komponen siap sebelum menjalankan skenario serangan.

---

## A. Urutan Startup VM

```
1. wazuh-server          (paling lama butuh waktu fully up)
2. suricata-ids
3. endpoint-linux-waz-plunk
4. metasploitable2
5. kali                  (terakhir, karena dia yang "menyerang")
```

Tunggu **1-2 menit** setelah `wazuh-server` menyala sebelum lanjut cek service-nya — Wazuh Manager, Indexer, dan Dashboard butuh waktu untuk fully initialize.

---

## B. Checklist Verifikasi

### B.1 — Wazuh Manager / Indexer / Dashboard

```bash
sudo systemctl status wazuh-manager
sudo systemctl status wazuh-indexer
sudo systemctl status wazuh-dashboard
```

✅ Ketiganya harus `active (running)`.

Buka browser ke `https://192.168.56.104` — pastikan dashboard terbuka normal, tidak muncul pesan *"server is not ready"*.

### B.2 — Status semua agent

```bash
sudo /var/ossec/bin/agent_control -lc
```

✅ Semua agent harus berstatus **Active**, bukan *Disconnected* atau *Never connected*.
✅ Catat **agent ID** masing-masing — bandingkan dengan ID yang digunakan di setiap `<active-response>` block pada `/var/ossec/etc/ossec.conf`. Jika ada agent dengan ID baru (biasanya karena hostname berubah), active-response yang menunjuk ke ID lama perlu diperbarui.

### B.3 — VM Suricata (cek tambahan khusus)

```bash
ip link show enp0s8                        # harus ada flag PROMISC
sudo systemctl status suricata              # harus active (running)
sudo systemctl status promisc-enp0s8.service # harus active (exited), status 0/SUCCESS
```

### B.4 — Konektivitas antar-VM

Dari Kali, ping ke setiap VM lain:
```bash
ping -c 2 192.168.56.104   # wazuh-server
ping -c 2 192.168.56.111   # suricata-ids (sesuaikan IP)
ping -c 2 192.168.56.106   # endpoint
ping -c 2 192.168.56.103   # metasploitable2 (sesuaikan IP)
```
✅ Semua harus `0% packet loss`.

### B.5 — Baseline dashboard

Buka dashboard Wazuh, pastikan tidak ada alert "nyasar" dari sesi lama yang bisa mengacaukan analisis sesi baru. Catat kondisi awal (jumlah alert, auth failure/success) sebagai baseline pembanding sebelum serangan dimulai.

---

## C. Troubleshooting — Dashboard "Not Ready" / Tidak Bisa Diakses

**Gejala:** browser menampilkan *"Wazuh server is not ready"* atau dashboard tidak bisa dibuka sama sekali.

1. Cek ketiga service inti:
   ```bash
   sudo systemctl status wazuh-manager
   sudo systemctl status wazuh-indexer
   sudo systemctl status wazuh-dashboard
   ```
2. **Jika `wazuh-indexer` berstatus `failed`** (terutama `code=killed, signal=KILL`):
   ```bash
   sudo systemctl restart wazuh-indexer
   ```
   Tunggu 1-2 menit (indexer berbasis Java/OpenSearch, startup-nya tidak instan).
3. Jika indexer tetap gagal, cek apakah ada proses yang mati karena OOM (kehabisan RAM):
   ```bash
   sudo dmesg | grep -i "killed process\|out of memory" | tail -5
   ```
   Jika ada, pertimbangkan menambah RAM VM atau mematikan VM lain yang tidak sedang dipakai.
4. Setelah indexer sehat, coba akses dashboard lagi.

---

## D. Troubleshooting — `wazuh-manager` Berstatus `failed`

**Gejala:** `systemctl status wazuh-manager` menunjukkan `Active: failed (Result: timeout)`, meski beberapa sub-proses lama masih terlihat berjalan.

**Penyebab umum:** proses lama tidak mati bersih setelah VM idle lama, sehingga restart berikutnya timeout.

**Langkah perbaikan (coba berurutan):**

1. **Restart standar dulu:**
   ```bash
   sudo systemctl restart wazuh-manager
   ```
   Jika log menunjukkan semua proses *"already running"* (bukan *"Started ..."*), ini bukan restart bersih — lanjut ke langkah 2.

2. **Restart paksa, bersih:**
   ```bash
   sudo systemctl stop wazuh-manager
   sudo pkill -9 -f wazuh
   sleep 3
   sudo systemctl start wazuh-manager
   sudo systemctl status wazuh-manager
   ```
   ⚠️ **Perhatian:** `pkill -9 -f wazuh` mencocokkan *semua* proses yang mengandung kata "wazuh", termasuk `wazuh-indexer` yang merupakan service terpisah. Setelah langkah ini, **selalu cek ulang status indexer juga** (lihat bagian C).

3. **Alternatif yang lebih aman (disarankan ke depannya):** gunakan `wazuh-control` yang menangani seluruh daemon secara "proper", tanpa risiko collateral ke service lain:
   ```bash
   sudo /var/ossec/bin/wazuh-control restart
   ```
   Catatan: setelah ini, `systemctl status wazuh-manager` kadang menampilkan info basi (timestamp lama, cgroup kosong) karena proses dikelola di luar jalur systemd langsung. Verifikasi proses yang sebenarnya jalan dengan:
   ```bash
   ps aux | grep wazuh
   ```
   PID yang baru dan log `"Started wazuh-execd..."` dsb (bukan `"already running"`) menandakan restart benar-benar bersih.

---

## E. Troubleshooting — Active Response Tidak Berjalan

**Gejala:** rule terdeteksi (muncul di dashboard), tapi IP attacker tidak ter-block; tidak ada entry baru di `active-responses.log`.

**Langkah diagnosis berurutan:**

1. **Cek apakah command active-response benar-benar dikirim manager:**
   ```bash
   sudo grep "100002" /var/ossec/logs/ossec.log | tail -10
   ```
   (sesuaikan nomor rule). Jika kosong, kemungkinan manager tidak sehat (lihat bagian D) atau rule tidak benar-benar ter-trigger.

2. **Cek log di endpoint (bukan di manager!)** — eksekusi `firewall-drop` terjadi di agent, bukan manager:
   ```bash
   sudo tail -30 /var/ossec/logs/active-responses.log
   ```
   Perhatikan apakah ada entry dengan timestamp terbaru dan status `"continue"` (sukses) vs `"Aborted"` (gagal/ditolak).

3. **Cek kecocokan `agent_id`:**
   ```bash
   sudo /var/ossec/bin/agent_control -lc
   ```
   Bandingkan ID agent yang aktif dengan `<agent_id>` di `<active-response>` block pada `/var/ossec/etc/ossec.conf` (manager). **Hostname yang berubah akan membuat Wazuh mendaftarkan agent baru dengan ID berbeda** — active-response yang masih menunjuk ID lama tidak akan pernah sampai ke agent yang benar. Jika tidak cocok:
   ```bash
   sudo nano /var/ossec/etc/ossec.conf
   # ubah <agent_id>LAMA</agent_id> menjadi ID yang aktif sekarang
   sudo /var/ossec/bin/wazuh-control restart
   ```

4. **Verifikasi hasil setelah perbaikan:**
   ```bash
   # dari attacker:
   ping -c 4 <IP_ATTACKER_DICEK_DARI_VICTIM>

   # dari endpoint:
   sudo iptables -L -n | grep <IP_ATTACKER>
   sudo tail -10 /var/ossec/logs/active-responses.log
   ```

---

## F. Troubleshooting — Suricata Tidak Menangkap Traffic Pihak Ketiga

**Gejala:** `ip a` / ping ke Suricata sendiri berhasil tercatat, tapi traffic antar-VM lain (misal Kali → Metasploitable2) tidak muncul di `eve.json`.

1. **Cek flag PROMISC di interface:**
   ```bash
   ip link show enp0s8
   ```
   Jika tidak ada flag `PROMISC` meski VirtualBox Settings sudah "Allow All" — setting VirtualBox saja **tidak cukup**, perlu diaktifkan eksplisit di level OS:
   ```bash
   sudo ip link set enp0s8 promisc on
   sudo systemctl restart suricata
   ```

2. **Pastikan ini permanen** (tidak hilang setelah reboot) — cek service pendukung:
   ```bash
   sudo systemctl status promisc-enp0s8.service
   ```
   Jika service ini tidak ada/tidak aktif, lihat bagian G untuk membuatnya.

3. **Validasi di level kernel dulu (lepas dari Suricata)** jika masih ragu:
   ```bash
   sudo tcpdump -i enp0s8 icmp -n
   ```
   Lalu generate ping dari VM lain ke VM lain (bukan ke Suricata). Jika `tcpdump` menangkapnya, promiscuous mode sudah benar di level kernel — masalah selanjutnya ada di konfigurasi Suricata itu sendiri (lanjut ke langkah 4).

4. **Cek Suricata benar-benar jalan tanpa error, dan sebagai service (bukan foreground manual):**
   ```bash
   sudo journalctl -u suricata -n 50 --no-pager
   sudo systemctl status suricata
   ```
   ⚠️ Jangan jalankan Suricata manual di foreground (`sudo suricata -c ... -i ...`) lalu mengetik command lain di terminal yang sama — ini akan mengirim sinyal interrupt dan mematikan Suricata secara tidak sengaja. Gunakan `systemctl start suricata` agar berjalan di background secara independen dari sesi terminal manapun.

---

## G. Membuat Ulang Service Promiscuous Mode Permanen (jika hilang)

Jika `promisc-enp0s8.service` tidak ada (misal di VM Suricata yang baru / rebuild):

```bash
sudo nano /etc/systemd/system/promisc-enp0s8.service
```

Isi:
```ini
[Unit]
Description=Enable promiscuous mode on enp0s8 for Suricata
After=network-online.target
Wants=network-online.target
Before=suricata.service

[Service]
Type=oneshot
ExecStart=/sbin/ip link set enp0s8 promisc on
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

Aktifkan:
```bash
sudo systemctl daemon-reload
sudo systemctl enable promisc-enp0s8.service
sudo systemctl start promisc-enp0s8.service
```

---

## H. Troubleshooting — Suricata "No Rules Loaded"

**Gejala:** warning `No rule files match the pattern ...` saat Suricata start.

```bash
sudo suricata-update
sudo systemctl restart suricata
```

Verifikasi:
```bash
sudo ls -la /var/lib/suricata/rules/
sudo systemctl status suricata   # pastikan tidak ada warning rule saat start
```

---

## I. Troubleshooting — Proses Suricata "Nyangkut" (Cluster-ID Conflict)

**Gejala:** error `af-packet: fanout not supported by kernel: Kernel too old or cluster-id 99 already in use.`

**Penyebab umum:** ada instance Suricata lama yang di-suspend (`Ctrl+Z`) alih-alih dimatikan (`Ctrl+C`), sehingga masih memegang resource capture.

```bash
jobs
ps aux | grep suricata
sudo kill -9 <PID1> <PID2> <PID3> ...   # kill eksplisit per-PID, pkill biasa kadang tidak cukup
ps aux | grep suricata                   # pastikan bersih (hanya proses grep yang tersisa)
sudo suricata -c /etc/suricata/suricata.yaml -i enp0s8
```

> Pelajaran: hindari `Ctrl+Z` untuk "menghentikan" proses foreground — itu hanya men-suspend, bukan mematikan. Gunakan `Ctrl+C`, atau jalankan Suricata sebagai service (`systemctl start suricata`) agar tidak terikat ke sesi terminal sama sekali.

---

## J. Ringkasan Cepat — "Semua Terasa Aneh, Mulai Dari Mana?"

Urutan diagnosis umum ketika banyak hal terasa tidak beres sekaligus (biasanya setelah lab idle lama):

1. `sudo systemctl status wazuh-manager wazuh-indexer wazuh-dashboard` di Wazuh Server
2. `sudo /var/ossec/bin/agent_control -lc` — cek semua agent aktif & ID-nya sesuai
3. Cek `ossec.conf` di manager — pastikan semua `<agent_id>` masih relevan dengan agent yang aktif
4. Di VM Suricata — cek `ip link show enp0s8` (PROMISC) dan `systemctl status suricata`
5. Ping test menyeluruh antar-VM dari Kali
6. Baru mulai skenario serangan setelah semua di atas hijau/normal
