# Paket Perbaikan Siap-Tempel — LMS Moodle 182.253.110.157

> Semua konfigurasi di bawah menutup temuan di `laporan-keamanan-website-sekolah.md`.
> Urutan eksekusi = urutan risiko. Lakukan backup sebelum mengubah konfigurasi.

---

## 1. 🔴 HTTPS + Paksa Redirect (menutup penyadapan kredensial)

Server Anda OpenLiteSpeed. Pasang sertifikat gratis:

```bash
# Jika pakai CyberPanel:
# SSL → Issue SSL → pilih domain → Let's Encrypt

# Jika OpenLiteSpeed manual (webserver harus punya domain, bukan raw IP):
# 1. Arahkan DNS A record (mis. lms.smkn1batang.sch.id → 182.253.110.157)
# 2. Terbitkan sertifikat:
certbot certonly --webroot -w /usr/local/lsws/Example/html -d lms.smkn1batang.sch.id
# 3. OLS WebAdmin → Listeners → tambah SSL Listener port 443,
#    isi cert/key: /etc/letsencrypt/live/lms.smkn1batang.sch.id/{cert,privkey}.pem
# 4. Listener port 80 → Rewrite → Enable Rewrite, Rewrite Rules:
```

```apache
RewriteCond %{HTTPS} off
RewriteRule ^(.*)$ https://%{HTTP_HOST}/$1 [R=301,L]
```

Lalu di `moodledata` (via admin): **Site administration → Security → HTTP security**
- ✅ Use HTTPS for logins → ubah menjadi *always* (`$CFG->cookiesecure` = true)
- Setelah aktif, logout semua sesi lama: `php admin/cli/purge_caches.php`

---

## 2. 🔴 Kunci phpMyAdmin (menutup gerbang database)

Pilih **satu** (urut dari terbaik):

**A. Hapus / nonaktif sementara (paling aman):**
```bash
mv /usr/local/lsws/Example/html/phpmyadmin /root/phpmyadmin-offline
# akses DB cukup via SSH: mysql -u moodleuser -p moodle
```

**B. IP allowlist + basic auth** — buat `/usr/local/lsws/Example/html/phpmyadmin/.htaccess`:
```apache
<IfModule mod_authz_core.c>
  Require ip 182.253.110.157 192.168.0.0/16 <IP_KANTOR/SEKOLAH>
</IfModule>
AuthType Basic
AuthName "Admin Only"
AuthUserFile /etc/lsws/.htpasswd
Require valid-user
```
```bash
htpasswd -c /etc/lsws/.htpasswd adminsekolah
```

**C. Jika dibiarkan terbuka (tidak disarankan):** pastikan user MySQL phpMyAdmin BUKAN `root`, password unik ≥20 karakter, dan `AllowNoPassword = false` di `config.inc.php`.

---

## 3. 🟠 Matikan Directory Listing di 20 Direktori (menutup peta source code)

Buat `.htaccess` di webroot — OpenLiteSpeed membacanya jika rewrite aktif:

```apache
# matikan autoindex global
Options -Indexes

# blokir file sensitif
<FilesMatch "\.(sql|zip|tar|gz|bak|old|orig|save|swp|log|ini|lock|json|yml|dist|txt)$">
  Require all denied
</FilesMatch>
<FilesMatch "(^\.|config\.php|install\.php|config-dist)">
  Require all denied
</FilesMatch>
```

> Catatan: rule di atas ikut memblokir `*.txt` publik seperti `lib/upgrade.txt` & `INSTALL.txt` — itu memang tujuannya. Jika ada file txt yang harus publik, whitelisting per-file.

Alternatif permanen di OLS WebAdmin: **Virtual Hosts → Example → Context → add Static Context** dengan `Options -Indexes`, atau hapus modul autoindex.

---

## 4. 🟠 Upgrade Moodle 4.4.2 → 4.5 LTS (menutup CVE publik, termasuk RCE calculated questions)

```bash
# 1. Backup penuh dulu
mysqldump -u moodleuser -p --single-transaction moodle > /root/moodle-db-$(date +%F).sql
tar czf /root/moodle-web-$(date +%F).tar.gz /usr/local/lsws/Example/html   # simpan DI LUAR webroot!

# 2. Pakai CLI updater resmi Moodle
cd /path/ke/moodle
sudo -u <user-web> php admin/cli/upgrade.php --non-interactive

# 3. Update tema academi dari vendor (LMSACE) ke versi terbaru
# 4. Purge cache
sudo -u <user-web> php admin/cli/purge_caches.php
```

- Jalankan di jendela sepi (malam/akhir pekan), uji login + 1 kursus setelahnya.
- Target akhir: jalur 4.5 LTS terbaru (atau 4.4 terakhir bila terpaksa) — 4.4.2 punya CVE RCE publik.

---

## 5. 🟡 Rapikan Identitas & Info Bocor

- Ganti email admin & email reset password dari Gmail umum → `admin@smkn1batang.sch.id`, aktifkan **2FA di Gmail Workspace** untuk akun itu.
- Footer: hapus/tukar placeholder sosial media (`yourtwittername`, `yourfacebookid`).
- Hapus direktori `/docs/` (manual OpenLiteSpeed) dari webroot.
- `Site admin → Security → Site policies`:
  - ✅ Limit concurrent logins
  - ✅ Lockout setelah 5 gagal × 15 menit
  - ✅ Password policy: min 10 karakter + angka + huruf besar/kecil
  - Nonaktifkan **webservices** yang tidak dipakai (`Site admin → Server → Web services`), atau batasi protokol.

---

## 6. 🟡 Kebijakan Backup (mencegah kebocoran mendatang)

```bash
# contoh cron harian: simpan di luar webroot + rotasi 14 hari
30 1 * * * mysqldump -u moodleuser -p'***' --single-transaction moodle | gzip > /root/backups/db-$(date +\%F).sql.gz
45 1 * * * find /root/backups -name "*.gz" -mtime +14 -delete
```

- **Aturan mutlak**: tidak ada `.sql/.zip/.tar.gz` di dalam webroot — directory listing akan menampilkannya dalam hitungan detik.
- Simpan salinan mingguan ke penyimpanan eksternal (cloud drive sekolah).

---

## 7. Verifikasi Setelah Perbaikan (5 menit)

```bash
curl -I http://lms.smkn1batang.sch.id | grep -i location          # harus redirect ke https
curl -sk https://lms.smkn1batang.sch.id -o /dev/null -w "%{http_code}\n"   # 200
curl -sk https://lms.smkn1batang.sch.id/backup/ | grep "Index of" # harus kosong
curl -sk -o /dev/null -w "%{http_code}\n" https://lms.smkn1batang.sch.id/phpmyadmin/  # 401/403
curl -sk -o /dev/null -w "%{http_code}\n" https://lms.smkn1batang.sch.id/lib/upgrade.txt # 403
```

---

## Checklist Ringkas

| # | Aksi | Effort | Status |
|---|------|--------|--------|
| 1 | HTTPS + redirect + `cookiesecure` | 2 jam | ☐ |
| 2 | Hapus/kunci phpMyAdmin | 30 mnt | ☐ |
| 3 | `Options -Indexes` + FilesMatch | 15 mnt | ☐ |
| 4 | Upgrade Moodle + tema | ½ hari | ☐ |
| 5 | Email domain + 2FA + lockout + hapus /docs/ | 1 jam | ☐ |
| 6 | Cron backup di luar webroot | 30 mnt | ☐ |
| 7 | Verifikasi curl | 5 mnt | ☐ |
