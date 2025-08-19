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

