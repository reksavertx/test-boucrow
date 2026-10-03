# Laporan Uji Keamanan Website LMS Sekolah

| | |
|---|---|
| **Target** | `http://182.253.110.157/` (LMS Moodle, sitename "SS") |
| **Identitas terkait** | SMK Negeri 1 Batang (`smkn1batang.sch.id`) — sesuai tautan & email di footer situs |
| **Tanggal uji** | 2 Oktober 2026 |
| **Metode** | Rekon pasif + probing GET-only (tanpa payload injeksi, tanpa brute-force, tanpa percobaan login, tanpa perubahan data) |
| **Otorisasi** | Izin internal dari pengelola (pelapor adalah pegawai sekolah) |
| **Pembuat** | Uji keamanan internal — bukan pentest tersertifikasi |

> ⚠️ **Catatan**: Ini bukan pengujian penetrasi menyeluruh. Uji eksploitasi aktif (LFI, SQLi, brute-force) **tidak dilakukan** dan sebaiknya hanya dijalankan dengan izin tertulis resmi dari kepala sekolah/yayasan.

---

## 1. Ringkasan Eksekutif

Situs tersebut adalah **Moodle LMS versi 4.4.2** yang berjalan di atas **LiteSpeed Web Server**, dan **sama sekali tidak menggunakan HTTPS** (port 443 tertutup). Ditemukan beberapa masalah serius yang berpotensi menyebabkan **kebocoran data dan pengambilalihan sistem** — persis jenis celah yang bisa dimanfaatkan pihak tak bertanggung jawab untuk mengubah database, sebagaimana insiden yang pernah terjadi.

| # | Temuan | Risiko | Status |
|---|--------|--------|--------|
| 1 | Tidak ada HTTPS sama sekali — kredensial dikirim plaintext | 🔴 **Critical** | Terkonfirmasi |
| 2 | phpMyAdmin 5.2.2 terekspos publik (`/phpmyadmin/`) | 🔴 **High** | Terkonfirmasi |
| 3 | Directory listing aktif di banyak direktori (autoindex LiteSpeed) | 🟠 **High** | Terkonfirmasi |
| 4 | Versi Moodle 4.4.2 (Agustus 2024) terpapar publik + tertinggal patch keamanan | 🟠 **Medium** | Terkonfirmasi |
| 5 | Informasi identitas sensitif terekspos di halaman publik | 🟡 **Medium** | Terkonfirmasi |
| 6 | Endpoint webservice/API terbuka (`login/token.php`, REST) | 🟡 **Low–Medium** | Terkonfirmasi |
| 7 | File standar Moodle terekspos (`upgrade.txt`, `composer.json`, `INSTALL.txt`) | 🟢 **Low** | Terkonfirmasi |

---

## 2. Detail Temuan

### 2.1 🔴 CRITICAL — Tidak Ada HTTPS (Semua Kredensial Berjalan Plaintext)

```
curl https://182.253.110.157/  →  gagal, port 443 tertutup
curl http://182.253.110.157/   →  200 OK (HTTP polos)
```

- Seluruh login (siswa, guru, **admin**) dikirim dalam bentuk teks yang bisa dibaca siapa pun yang bisa menyadap jalur jaringan.
- Cookie `MoodleSession` dikirim tanpa flag `Secure` (tidak mungkin diset karena tak ada TLS).
- **Skenario penyalahgunaan**: siapa pun di jaringan WiFi sekolah yang sama (murid, tamu) dapat menyadap kredensial guru/admin dengan tool sniffing sederhana → login sebagai admin → mengubah data nilai/murid langsung dari UI Moodle, atau membuka phpMyAdmin.
- **Skenario kedua**: pendaftar PPDB yang jahil cukup melakukan ARP spoofing di jaringan sekolah untuk mencuri sesi admin.

**Rekomendasi:**
1. Pasang SSL gratis (Let's Encrypt / ZeroSSL) melalui panel LiteSpeed/CyberPanel, aktifkan **force redirect HTTP→HTTPS**.
2. Set cookie Moodle dengan `cookiesecure = true` di `config.php`.
3. Pertimbangkan HSTS setelah HTTPS stabil.

---

### 2.2 🔴 HIGH — phpMyAdmin Terekspos Publik

```
GET /phpmyadmin/         → 200 (halaman login terbuka untuk publik)
GET /phpmyadmin/README   → 200, "Version 5.2.2"
GET /phpmyadmin/setup/   → 200 (wizard konfigurasi dapat diakses)
```

- phpMyAdmin adalah **gerbang langsung ke seluruh database** Moodle (tabel `mdl_user` berisi nama, email, username, password hash seluruh siswa & guru).
- Halaman login terbuka untuk internet publik → bisa di-brute-force / credential-stuffing tanpa henti.
- Wizard `/phpmyadmin/setup/` yang seharusnya hanya untuk instalasi juga dapat diakses.
- Versi terpapar (5.2.2) — walau saat ini bukan versi yang rentan, paparan versi mempermudah penyerang menunggu/mencocokkan CVE masa depan.

**Skenario penyalahgunaan**: password MySQL yang lemah / dipakai ulang dari akun lain → full akses database → mengubah data pendaftar, nilai, atau menanam halaman phishing di database.

**Rekomendasi:**
1. **IP allowlist**: batasi `/phpmyadmin/` hanya dari jaringan kantor/server admin (aturan LiteSpeed ACL atau `.htaccess` `Require ip`).
2. Atau tambahkan lapisan **HTTP Basic Auth** di depan login phpMyAdmin.
3. Idealnya nonaktifkan phpMyAdmin sepenuhnya; gunakan akses SSH/CLI untuk maintenance.
4. Pastikan password user MySQL **bukan** password yang sama dengan akun lain, dan user phpMyAdmin bukan `root`.

---

### 2.3 🟠 HIGH — Directory Listing Aktif (Autoindex LiteSpeed)

Direktori tanpa `index.php` menampilkan **daftar seluruh isinya** ke publik:

```
GET /backup/          → "Index of /backup/"  (21 item source code PHP terlihat)
GET /question/        → listing terbuka
GET /theme/academi/   → listing terbuka (struktur tema komersial terpapar)
GET /report/          → listing 15+ subdirektori
GET /local/, /repository/, /admin/tool/, /backup/util/ → listing terbuka
```

- Penyerang dapat **memetakan seluruh struktur source code** tanpa tool khusus.
- **Risiko terbesar**: jika suatu saat ada file backup database (`.sql`, `.zip`, `.bak`), log, atau file hasil export yang tersimpan di dalam webroot, file itu **langsung terlihat dan dapat diunduh** oleh siapa pun. Ini adalah penyebab kebocoran data paling umum di server sekolah.
- Direktori yang punya `index.php` (`/mod/`, `/enrol/`, `/grade/`, dll.) tidak menampilkan listing — bukan karena kebijakan server, tapi kebetulan struktur aplikasi.

**Rekomendasi:**
1. Matikan autoindex di LiteSpeed (Index Files → hapus autoindex, atau `.htaccess` berisi `Options -Indexes` — LiteSpeed membaca `.htaccess`).
2. **Jangan pernah menyimpan** file `.sql`, `.zip`, `.bak`, `.log` di dalam webroot. Simpan backup di luar webroot atau di storage terpisah.
3. Tambahkan rule blokir akses langsung ke ekstensi sensitif:
   ```apache
   <FilesMatch "\.(sql|zip|bak|log|ini|old)$">
     Require all denied
   </FilesMatch>
   ```

---

### 2.4 🟠 MEDIUM — Versi Moodle Terpapar & Tertinggal Patch Keamanan

```
GET /lib/upgrade.txt → 200, entri teratas "=== 4.4.2 ==="
Link mobile app      → version=2024042202 (build Moodle 4.4.2, Agustus 2024)
```

- Moodle 4.4.2 dirilis **Agustus 2024** — per laporan ini sudah hampir 2 tahun tertinggal dari patch keamanan berkala Moodle.
- Setelah 4.4.2, Moodle merilis perbaikan keamanan penting, termasuk **CVE-2024-43425** (RCE pada tipe soal *calculated*, diperbaiki di 4.4.3) dan berbagai perbaikan XSS/IDOR pada rilis berikutnya.
- Versi terpapar publik via `upgrade.txt` → penyerang tinggal mencocokkan CVE publik.
- Tema "academi" (LMSACE) berwatermark "Copyright 2017" → kemungkinan tema/juga core jarang diperbarui.

**Rekomendasi:**
1. **Upgrade Moodle ke versi stabil terbaru** (jalur 4.4 terakhir atau 4.5 LTS / 5.x) — prioritas tinggi karena RCE publik ada di versi di bawahnya.
2. Blokir akses publik ke `lib/upgrade.txt`, `composer.json`, `INSTALL.txt`, `README` via `.htaccess`.
3. Aktifkan notifikasi rilis keamanan Moodle untuk admin.

---

### 2.5 🟡 MEDIUM — Informasi Identitas Terekspos di Halaman Publik

Dari halaman depan saja (tanpa login), seorang penyerang sudah mendapat:

| Informasi | Lokasi | Kegunaan bagi penyerang |
|---|---|---|
| Email `smksatubatang@gmail.com` | Footer situs | Target **phishing/spear-phishing**; email Gmail umum dipakai sebagai email reset password admin → reset akun bisa dibajak jika Gmail itu lemah |
| Domain `smkn1batang.sch.id` | Tautan footer | Mempastikan situs milik sekolah → memungkinkan serangan lintas-sistem (deface domain utama, subdomain lain, DNS) |
| Sitename "SS" + branding LMSACE "Copyright 2017" | Judul & footer | Mengonfirmasi instalasi tidak dikustomisasi/diperbarui |
| Akses via raw IP tanpa domain | URL | Menandakan VPS mandiri; penyerang bisa cek host lain di IP/ASN yang sama |
| Tautan media sosial placeholder (`yourtwittername`, dst.) | Footer | Indikasi konfigurasi tema diselesaikan seadanya |

**Rekomendasi:**
- Gunakan email domain sekolah (`admin@smkn1batang.sch.id`) untuk akun admin & reset password, dengan 2FA aktif di Gmail Workspace-nya.
- Sembunyikan/tidak perlu menghapus — tapi sadari bahwa email umum + password lemah = kombinasi pembajakan.

---

### 2.6 🟡 LOW–MEDIUM — Endpoint API Terbuka

```
GET /login/token.php                → 200 {"error":"missingparam"}  (aktif)
GET /webservice/rest/server.php     → 200 respon XML "invalidtoken" (aktif)
```

- Ini endpoint resmi Moodle (untuk aplikasi mobile), **tetapi** juga jalur favorit untuk *password spraying* otomatis karena menerima JSON dan tidak memuat halaman penuh.
- Pastikan di Admin Moodle: **Site admin → Security → Webservices** hanya mengaktifkan layanan yang benar-benar dipakai, dan **lockout login** (Security → Site policies) diaktifkan (mis. kunci setelah 5 gagal).

---

### 2.7 🟢 LOW — File Standar Terekspos

| Path | Status | Dampak |
|---|---|---|
| `/lib/upgrade.txt` | 200 (184 KB) | Mengonfirmasi versi 4.4.2 |
| `/composer.json` | 200 | Konfirmasi requirement PHP >= 8.1, struktur proyek |
| `/INSTALL.txt` | 200 | Panduan instalasi default — konfirmasi instalasi standar |
| `/phpmyadmin/README` | 200 | Versi phpMyAdmin 5.2.2 |

Semua low-level, tapi bagian dari "peta" yang mempermudah penyerang. Blokir via `.htaccess`.

---

## 3. Hal "Sepele" yang Bisa Dipakai Penyerang

1. **Footer menampilkan email Gmail umum** → phising reset password admin.
2. **`/backup/` dan direktori lain bisa dilihat isinya** → cukup menunggu admin lupa menyimpan file `.sql` hasil export di situ (praktik sangat umum) → seluruh database siswa terunduh.
3. **Port 443 mati** → murid di WiFi sekolah bisa menyadap password guru dengan plugin browser/tools sniffing.
4. **Versi software terpapar di 3 tempat berbeda** (upgrade.txt, link mobile, README phpMyAdmin) → penyerang tak perlu menebak; tinggal pakai exploit publik yang cocok.
5. **`/phpmyadmin/setup/` terbuka** → menandakan konfigurasi server web dibiarkan default pabrik.
6. **Tidak ada `robots.txt` maupun `security.txt`** → tidak ada kanal resmi pelaporan kerentanan; peneliti baik tak bisa melapor, penyerang tak dihalangi apa pun.
7. **Login page standar Moodle tanpa proteksi tambahan** (CAPTCHA tidak terlihat) → cocok untuk credential stuffing massal dengan daftar akun `fullname`/NISN yang mudah ditebak dari pola sekolah Indonesia.

---

## 4. Yang Sudah Aman (Temuan Negatif)

| Pemeriksaan | Hasil |
|---|---|
| `/.env`, `/.git/`, `/adminer.php`, `/.DS_Store`, `/phpinfo.php` | 404 — tidak ada |
| `/config.php` | 200 tapi body kosong — tidak bocor kredensial |
| `/install.php` | Redirect — tidak bisa reinstall |
| `/admin/cron.php` | "internet access disabled" — cron web sudah dimatikan ✅ |
| `/user/profile.php?id=1`, `/userpix/` | Redirect ke login — enumerasi user admin terbatas ✅ |
| `/admin/index.php` | Redirect ke login ✅ |
| Cookie `MoodleSession` | `HttpOnly` ✅ (tapi tanpa `Secure`, `SameSite` belum diset) |
| Header `X-Frame-Options: sameorigin` | Ada ✅ (anti clickjacking dasar) |
| Versi server di header | Tidak ditampilkan ✅ |

---

## 5. Skenario Serangan Nyata (Alur Lengkap)

Berdasarkan temuan di atas, inilah jalur termudah seorang "hacker sewaan" mengubah database sekolah:

```
1. Bukti sudah ada di halaman publik: versi Moodle 4.4.2, phpMyAdmin terbuka, email admin Gmail.
2. Jalur A (termudah): menyadap di jaringan sekolah → karena tak ada HTTPS, password admin/guru tertangkap → login → ubah data via UI.
3. Jalur B: brute-force/credential-stuffing di /phpmyadmin/ atau /login/token.php (tak dibatasi CAPTCHA) → akses database penuh.
4. Jalur C: eksploitasi CVE Moodle 4.4.x (RCE calculated question, butuh akun guru/teacher) → shell → config.php → database.
5. Jalur D (paling pasif): menunggu admin melakukan export/backup .sql di webroot → directory listing menampilkannya → unduh semua data.
```

---

## 6. Prioritas Perbaikan

| Prioritas | Aksi | Effort |
|---|---|---|
| 🔴 1 | Aktifkan HTTPS + force redirect + cookie `Secure` | 1–2 jam |
| 🔴 2 | Batasi `/phpmyadmin/` (IP allowlist / basic auth / hapus) | 30 menit |
| 🟠 3 | Matikan directory listing (`Options -Indexes`) | 15 menit |
| 🟠 4 | Upgrade Moodle 4.4.2 → versi stabil terbaru + tema academi | 1 hari (test dulu) |
| 🟠 5 | Kebijakan: larangan menyimpan backup/export di webroot | kebijakan |
| 🟡 6 | 2FA untuk akun admin, aktifkan login lockout, ganti password admin & MySQL | 1 jam |
| 🟡 7 | Blokir akses ke `*.txt`/`composer.json` di webroot + ekstensi `.sql/.zip/.bak` | 30 menit |
| 🟢 8 | Tambah `security.txt`, `robots.txt`, dokumentasi insiden | 30 menit |

---

## 7. Checklist Pengujian Lanjutan (Butuh Izin Tertulis Resmi)

Hal-hal berikut **tidak dijalankan** dalam audit ini karena bersifat eksploitasi aktif. Jalankan hanya dengan otorisasi tertulis:

- [ ] LFI: uji parameter GET/POST Moodle & plugin tema academi dengan wordlist traversal (`../../etc/passwd`, `php://filter`) — gunakan `ffuf`/`wfuzz` kecepatan rendah.
- [ ] SQLi: uji parameter `id`/`courseid` dengan `sqlmap --level=1 --risk=1` (read-only).
- [ ] Bruteforce phpMyAdmin (dengan izin) untuk mengukur ada/tidaknya rate-limit.
- [ ] Enumerasi username via respons login/index.php dan forgot_password.
- [ ] Scan plugin tambahan di `/mod/`, `/local/`, `/theme/academi/` terhadap CVE tema komersial LMSACE.
- [ ] Pemindaian port lanjutan (dengan izin pemilik server) untuk layanan non-HTTP (SSH, MySQL eksternal, panel).

---

*Laporan ini dibuat sebagai bagian dari upaya internal perlindungan data murid. Distribusikan hanya ke pihak berwenang di sekolah.*
