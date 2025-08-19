### How To Configure
**script deteksi DDoS + notifikasi ke bot Telegram**.
Logikanya: script memantau jumlah koneksi per IP → jika melebihi ambang batas (misal 200 koneksi), maka kirim notifikasi ke Telegram.

---

### 🔹 1. Buat Bot Telegram

1. Buka Telegram, cari `@BotFather`.
2. Buat bot baru dengan `/newbot`.
3. Catat **TOKEN** dari BotFather.
4. Tambahkan bot ke grup (kalau mau kirim ke grup) atau gunakan chat pribadi.
5. Ambil **CHAT\_ID** dengan cara buka di browser:

   ```
   https://api.telegram.org/botTOKEN/getUpdates
   ```

   (ganti `TOKEN` dengan token botmu).

---

### 🔹 2. Script Bash Notifikasi DDoS

Simpan script ini, misalnya `/root/ddos-monitor.sh`:

```bash
#!/bin/bash

# Konfigurasi Telegram
TOKEN="ISI_TOKEN_BOTMU"
CHAT_ID="ISI_CHAT_ID"

# Batas koneksi per IP (misal 200 koneksi)
LIMIT=200

# Cari IP yang melebihi batas
ATTACKERS=$(netstat -ntu 2>/dev/null | awk '{print $5}' | cut -d: -f1 | \
            sort | grep -v '^$' | uniq -c | sort -nr | awk -v limit=$LIMIT '$1 > limit {print $2 " (" $1 " koneksi)"}')

if [ ! -z "$ATTACKERS" ]; then
    MESSAGE="⚠️ *DDOS ALERT* ⚠️%0A%0ADitemukan IP mencurigakan:%0A$ATTACKERS"
    curl -s "https://api.telegram.org/bot$TOKEN/sendMessage" \
         -d chat_id="$CHAT_ID" \
         -d text="$MESSAGE" \
         -d parse_mode="Markdown"
fi
```

---

### 🔹 3. Cara Jalankan Otomatis

1. Jadikan script executable:

   ```bash
   chmod +x /root/ddos-monitor.sh
   ```

2. Tambahkan ke cronjob (misal cek tiap 1 menit):

   ```bash
   crontab -e
   ```

   Tambahkan baris:

   ```
   * * * * * /root/ddos-monitor.sh
   ```

---

### 🔹 4. Hasil

Kalau ada IP yang koneksinya melebihi **200** dalam 1 menit, bot Telegrammu akan mengirim notifikasi:

```
⚠️ DDOS ALERT ⚠️
Ditemukan IP mencurigakan:
192.168.1.100 (350 koneksi)
203.0.113.50 (420 koneksi)
```

---

### Cara Atasi Serangan 

Mengatasi serangan **DDoS (Distributed Denial of Service)** tergantung seberapa besar serangan dan resource server/VPS yang kamu pakai. Saya jelaskan dari yang paling dasar sampai level advance:

---

### 🔹 1. Deteksi Awal

* **Cek trafik server** → gunakan `htop`, `iftop`, atau `nload` untuk melihat beban CPU, RAM, dan bandwidth.
* **Cek koneksi masuk** →

  ```bash
  netstat -ntu | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -n
  ```

  Ini menampilkan IP yang banyak melakukan koneksi.

---

### 🔹 2. Mitigasi di Level Server

* **Batasi koneksi per IP** (pakai iptables):

  ```bash
  iptables -A INPUT -p tcp --dport 80 -m connlimit --connlimit-above 50 -j DROP
  ```

  Artinya, 1 IP maksimal 50 koneksi ke port 80.

* **Rate limiting** (mencegah flood):

  ```bash
  iptables -A INPUT -p tcp --dport 80 -m limit --limit 25/s --limit-burst 100 -j ACCEPT
  iptables -A INPUT -p tcp --dport 80 -j DROP
  ```

* **Fail2ban** → otomatis blok IP yang terlalu banyak request tidak wajar.

  ```bash
  apt install fail2ban -y
  ```

* **Gunakan reverse proxy (Nginx/HAProxy)** dengan limit request:

  ```nginx
  limit_req_zone $binary_remote_addr zone=req_limit:10m rate=5r/s;
  server {
      location / {
          limit_req zone=req_limit burst=10 nodelay;
      }
  }
  ```

---

### 🔹 3. Perlindungan di Level Jaringan

* **Aktifkan firewall VPS** (UFW/iptables) untuk hanya membuka port penting.
* **Gunakan TCP SYN cookies** untuk mencegah SYN flood:

  ```bash
  sysctl -w net.ipv4.tcp_syncookies=1
  ```

---

### 🔹 4. Gunakan Layanan Pihak Ketiga

Kalau serangan besar, resource VPS biasanya tidak kuat. Solusi:

* **Cloudflare / DDoS Protection CDN** → gratis/bayar. Bisa filter trafik sebelum masuk ke server.
* **Anti-DDoS dari provider VPS** (Contoh: OVH, Path.net, Mitigator, dll).

---

### 🔹 5. Strategi Tambahan

* **Load Balancer**: Sebar trafik ke beberapa server.
* **Anycast DNS**: Gunakan DNS dengan proteksi DDoS.
* **Monitoring**: Gunakan tool seperti *Uptime Kuma, Grafana, atau Netdata* untuk deteksi dini.

---

👉 Jadi, kalau serangan kecil–menengah bisa diatasi dengan firewall + rate limit. Kalau serangan besar (ratusan Gbps), harus pakai layanan DDoS mitigation seperti Cloudflare atau provider dengan Anti-DDoS.

