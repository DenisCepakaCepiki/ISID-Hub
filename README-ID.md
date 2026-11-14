# Isekai Station ID (ISID Hub) v1.0.0

Platform distribusi script Lua/Luau **multi-produk** dengan manajemen lisensi dan autentikasi akun Roblox.
Website: https://isekaistationid.netlify.app

> Developed by Denis Cepaka, Claude & ChatGPT

Dokumen ini menjelaskan cara kerja, arsitektur, keamanan, dan **batasan** sistem secara jujur.
Pemasangan pertama kali: **`PANDUAN-INSTALASI-AWAL.md`**. Pemakaian harian setelah terpasang (dan upgrade): **`PANDUAN-SETELAH-TERPASANG.md`**. Perubahan: **`CHANGELOG.md`**. Dokumen lanjutan ada di folder `docs/`:

| Dokumen | Isi |
|---|---|
| `docs/DATABASE.md` | Skema SQLite, aturan data, migrasi, trigger |
| `docs/API.md` | Referensi 124 endpoint (dihasilkan otomatis dari kode) |
| `docs/SECURITY.md` | Model ancaman, kontrol keamanan, batasan |
| `docs/DEPLOYMENT.md` | Netlify, Cloudflare Tunnel, VS Code, operasional harian |
| `docs/BACKUP-RESTORE.md` | Backup, restore, backup `.env` terenkripsi |
| `docs/MIGRASI-V1-KE-V2.md` | Apa yang berubah dari V1 lama dan cara upgrade tanpa kehilangan data (upgrade dari V2.x: lihat `PANDUAN-SETELAH-TERPASANG.md`) |
| `docs/PANDUAN-TAMPILAN.md` | Menaruh wallpaper dan logo GIF, ukuran, warna |
| `docs/AUDIT-V1.md` | Hasil audit proyek V1 (temuan dan perbaikannya) |
| `docs/UJI-MANUAL.md` | Prosedur uji yang HARUS Anda jalankan sendiri (Roblox, Netlify, Cloudflare, executor) |
| `docs/LAPORAN-PENGUJIAN.md` | Laporan jujur: hasil test nyata dan yang belum terbukti |

---

## 1. Ringkasan sistem

```
Pengguna (Roblox executor / browser)
        |  HTTPS
        v
  Netlify (isekaistationid.netlify.app)
   - Halaman statis: landing, produk, dashboard pelanggan, admin
   - Function api-proxy : /api/*        -> backend
   - Function loader    : /loader/<slug> -> backend (HTML untuk browser, Lua untuk executor)
        |  HTTPS + header X-ISID-Gateway-Secret
        v
  Cloudflare Tunnel (Windows Service "Cloudflared" di PC admin, hostname permanen)
        |
        v
  Backend Node.js di PC admin (127.0.0.1:8787)
   - REST /api/v1   - SQLite data/isid.db   - Lua privat storage/private/<slug>/v<versi>.lua
```

* **Administrator hanya SATU.** Tidak ada peran owner/staff/support. Dijaga di database (indeks unik) dan di kode.
* **Produk 100% dinamis.** Tidak ada nama produk di kode sumber. Produk baru dibuat dari dashboard admin.
* **Backend tanpa paket npm.** Hanya modul bawaan Node.js 22.13+ (`node:http`, `node:sqlite`, `node:crypto`).
  Tidak ada `npm install` untuk backend. Router, validator, TOTP, dan rate limiter ditulis sendiri di `server/src/lib`.
* **PC admin harus menyala** (backend + tunnel berjalan) agar pelanggan bisa memuat script. Bila PC mati, halaman statis
  tetap tampil, dan executor menerima pesan "Server maintenance" beserta kontak Discord.

## 2. Prinsip terpenting: entitlement per produk

**Membeli satu produk tidak pernah membuka produk lain.** Ditegakkan di tiga lapis:

1. **Database.** `entitlements` punya `UNIQUE(user_id, product_id)`, dan kunci asing komposit
   `(license_id, user_id, product_id) -> licenses(id, user_id, product_id)` membuat lisensi produk A secara fisik tidak bisa dijadikan dasar akses produk B.
2. **Server.** Setiap `POST /api/v1/payload/request` memanggil pemeriksaan lisensi untuk **produk yang diminta**. Tidak ada cache akses dan tidak ada token yang berlaku lintas produk.
3. **Test otomatis.** Skenario A-G (di bagian 12) dijalankan di setiap `npm test` / task "ISID: Run Tests".

Status lisensi: `pending`, `active`, `expired`, `suspended`, `revoked`. Lisensi `active` yang tanggalnya sudah lewat
langsung dianggap `expired` saat itu juga. Status order: `pending`, `paid`, `rejected`, `cancelled`, `refunded`.

## 3. Struktur folder

```
ISID-Hub/
  package.json  netlify.toml  .env.example
  *.bat                    <- pintasan Windows (klik dua kali)
  .vscode/                 <- tasks.json, launch.json, settings.json (4 grup terminal)
  server/
    src/index.js app.js core.js config.js
    src/lib/               <- validator, crypto, TOTP, rate limit, router, logger, backup, klien Roblox
    src/db/                <- adapter SQLite + migrations/001..008
    src/services/          <- products, licenses, orders, auth, payload, admins, users, trials, discord, executionLog, sessions, audit, settings, backup, monitoring
    src/routes/            <- public, auth, me, control (lokal), admin/*
    src/lua/               <- generator + template bootstrap.lua
    scripts/               <- semua perintah CLI (.bat memanggil ini)
    tests/                 <- 332 test otomatis (+ mock Roblox untuk Lua)
  site/
    public/                <- index, product, maintenance, dashboard, legal, admin (HTML + JS tanpa inline script)
    public/assets/img/     <- tempat wallpaper.png dan logo.gif milik Anda (kosong di paket)
    netlify/functions/     <- api-proxy.mjs, loader.mjs
  seed/                    <- contoh script (TIDAK dimodifikasi dari V1)
  data/ storage/ backups/ logs/   <- diisi saat dipakai. JANGAN di-commit / dibagikan.
  docs/
```

## 4. Produk, versi, dan Lua

Dari **Admin -> Products -> + CREATE NEW PRODUCT** Anda mengisi nama, slug, deskripsi, ikon/gambar, kategori, tag, fitur, harga,
durasi lisensi, visibilitas, status, versi, changelog, dan file `.lua` (unggah dari komputer atau tempel). Setelah disimpan otomatis ada:
halaman produk, URL loader, penyimpanan Lua privat, riwayat versi, statistik, dan ketersediaan di toko.

* **Versi**: `draft` (boleh diganti/dihapus) atau `published`. **Versi published TIDAK BISA diubah** (immutable): dijaga trigger database
  (`trg_version_published_immutable`, `trg_payload_published_immutable`, dll.), bukan hanya kode. Perubahan = rilis versi baru.
* **Rollback**: jadikan versi published lama sebagai versi aktif. Versi draft tidak bisa dijadikan aktif.
* **Integritas**: SHA-256 dan ukuran tiap file disimpan di database. Dihitung ulang saat dikirim ke pelanggan, dan oleh `CHECK-DATABASE.bat`.
* **Penyimpanan**: `storage/private/<slug>/v<versi>.lua`. Tidak pernah disajikan lewat web statis. Mengganti payload draft menyalin isi lama ke `storage/trash/` lebih dulu (tidak ada data yang hilang).
* **Arsip**: produk yang diarsipkan menjadi `disabled`. Membatalkan arsip TIDAK otomatis mengaktifkan lagi: aktifkan eksplisit (disengaja, agar tidak terjual tanpa sengaja).
* **Hapus vs arsip**: produk yang pernah punya lisensi/order **tidak bisa dihapus** (server menolak, gunakan arsip). Produk tanpa riwayat boleh dihapus; file Lua-nya dipindah ke `storage/trash/`, bukan dihapus permanen.

## 5. `/loader/<slug>`: satu URL, dua respons

```lua
loadstring(game:HttpGet("https://isekaistationid.netlify.app/loader/<slug>"))()
```

| Pemanggil | Respons |
|---|---|
| **Browser** (navigasi dokumen) | Halaman HTML produk. Produk tidak ada: halaman 404. Maintenance: `/maintenance.html` (503). |
| **Executor** (`game:HttpGet`) | Lua bootstrap (selalu HTTP 200, supaya pesan tampil di layar Roblox, bukan error mentah). |

Pembedaan memakai `Sec-Fetch-Dest`/`Sec-Fetch-Mode`/`Accept`, ditambah petunjuk nama executor di `User-Agent`; semuanya **hanya untuk memilih tampilan** dan bisa dipalsukan. **Tidak ada header yang dipakai sebagai autentikasi**: izin selalu diputuskan backend (akun + lisensi).
Respons membawa `Cache-Control: no-store` dan `Vary` supaya CDN tidak mencampur halaman HTML dengan Lua.
Kasus khusus yang ditangani dan diuji: backend mati, slug tidak ada, produk disabled/draft, maintenance global, maintenance per produk.
Bootstrap bersifat publik dan aman dibagikan: tidak berisi secret maupun script produk. Script produk baru dikirim **setelah** akun dan lisensi produk itu lolos.

### Akun, login, dan pembuktian pemilik (v1.0.0)

Tidak ada pendaftaran mandiri dan tidak ada lagi verifikasi frasa di profil Roblox. Alurnya:

1. **Admin membuat akun** pelanggan (Admin -> Users -> *Create account*): username Roblox + password (atau password sementara acak yang tampil **sekali**).
2. Pelanggan login di `/dashboard` dengan **username Roblox + password ISID**, menyetujui Terms of Service dan Privacy Policy (pop-up "Saya setuju"), lalu **mengganti password sementara**.
3. Di Roblox, loader (`loadstring(game:HttpGet(".../loader/<slug>"))()`) memakai **kredensial yang sama**. Server memeriksa berurutan: **username pemain yang sedang menjalankan = username terdaftar** -> password -> UserId terdaftar -> password sementara sudah diganti -> **lisensi produk itu**. Setiap hasil dicatat (Admin -> Executions) dan dikirim ke webhook Discord.
4. Perangkat yang sudah login mendapat **device key** (hanya hash disimpan; 30 hari; jumlah perangkat dibatasi) sehingga eksekusi berikutnya tidak perlu mengetik ulang.

**Key trial gratis** (`ISID-XXXX-XXXX-XXXX`, dibuat admin) memberi akses sementara ke satu script **tanpa login** di loader; batasnya diatur per key (lama trial, jumlah klaim, masa berlaku, per IP).

Sistem **tidak pernah** meminta password Roblox atau cookie `.ROBLOSECURITY`, dan **tidak** memakai Roblox OAuth (butuh aplikasi OAuth terdaftar di Roblox). Password ISID adalah password milik ISID, bukan password Roblox: jangan mengisinya dengan password Roblox Anda.

**Batasan jujur:** username/UserId/executor/device yang dikirim loader dilaporkan oleh klien (script di sisi pemain) dan **bisa dipalsukan**. Pencocokan username terhadap akun yang login mencegah pemakaian kredensial yang tidak disengaja dan memudahkan pelacakan, tetapi pemain yang menguasai klien bisa berbohong; anggap log sebagai petunjuk, bukan bukti. Lihat `docs/SECURITY.md`.

### Executor yang diizinkan

Bootstrap melaporkan nama executor (`identifyexecutor()`) ke server. Di **Settings -> Executors**:

| Pengaturan | Arti |
|---|---|
| `enforce` (bawaan) | Executor di luar daftar **tidak menerima script** (pesan: "Executor ... is not supported" + kontak Discord) |
| `monitor` | Hanya dicatat; tidak ada yang diblokir. Pakai ini dulu untuk melihat nama persis yang dilaporkan executor pelanggan |
| `off` | Tidak diperiksa |

Daftar bawaan: **Delta, Real, Madium**. Tulis nama **persis seperti yang muncul** di tabel "Executors seen" (kapital tidak masalah). Tombol **Allow** di tabel itu menambahkan satu nama ke daftar.
**Batas jujur:** nama dilaporkan oleh klien, jadi ini hanya penyaring. Executor atau klien yang memalsukan nama tetap lolos. Lihat `docs/SECURITY.md`.

## 6. Pembayaran manual lewat Discord

Tidak ada payment gateway palsu dan tidak ada nomor bank/e-wallet/QRIS yang ditulis di kode. Alurnya:

1. Pelanggan login di `/dashboard`, membuat order (status `pending`). Halaman order menampilkan instruksi dan kontak Discord. (Alternatif tanpa order: pelanggan menghubungi Discord, admin langsung membuat akun + **Licenses -> Grant**.)
2. Pelanggan menghubungi admin lewat Discord (`denis.cepaka`, atau server https://discord.gg/isekai-station-id-703563738367655937), membayar sesuai instruksi admin, lalu (opsional) mengunggah bukti.
3. Admin memeriksa sendiri di **Orders & Payments**, lalu menekan **Approve**: order menjadi `paid` dan lisensi dibuat **hanya untuk produk di order itu**.

Persetujuan bersifat **idempoten dan transaksional**: menekan dua kali (atau dua tab sekaligus) hanya menghasilkan satu lisensi. Diuji paralel.
Webhook PUBLIC dapat mengumumkan "user telah membeli" (hanya username Roblox; bisa disembunyikan per akun atau global).
Kontak Discord disimpan di `system_settings` (bisa diubah dari **Settings**) dan tampil di landing, halaman produk, dashboard pelanggan, checkout,
instruksi pembayaran, dukungan, footer, dan halaman error yang relevan, dengan tombol *Contact Admin*, *Join ISID Discord*, *Copy Discord Username*, *Payment Support*.
Saat backend mati, Netlify Function tetap menampilkan kontak cadangan.

## 7. Dashboard

**Admin** (`/admin`, berbahasa Inggris): *Overview, Products, Users, Licenses, Orders & Payments, Executions, Trial Keys, Discord Webhooks, Lua Storage, Server Monitor, Database, Audit Logs, Settings, Security* (Security = password, 2FA, sesi admin).
Semua angka berasal dari data nyata; bagian yang tidak bisa dibaca ditulis **"unavailable"** (bukan angka karangan). Restart server dan restore database
meminta konfirmasi ulang password (dan kode 2FA bila aktif); restore juga wajib mengetik `RESTORE`.

* **Products**: buat produk, kartu **Update the script (new version)**, riwayat versi, **Roll back to vX**.
* **Users**: *Create account*, password sementara sekali tampil, *Reset password*, *Unlock*, profil, tab *Loader activity*.
* **Executions**: log setiap percobaan loader (username Roblox, UserId, executor, device, HWID ter-hash, IP, map/game); filter, ringkasan, refresh langsung; disimpan 90 hari (Settings).
* **Trial Keys**: buat, lihat, cabut/pulihkan, riwayat klaim.
* **Discord Webhooks**: ID Discord Anda, kartu OWNER (privat) dan PUBLIC, URL (terenkripsi), saklar, event, Send test, webhook kustom, pengumuman.

**Pelanggan** (`/dashboard`, **Indonesia (utama) dan English**): hanya produk yang dimiliki tampil lengkap dengan loader dan changelog; produk lain tampil sebagai belum dimiliki.
Ada daftar order dan status, notifikasi, perangkat, halaman Akun (ganti password), dan kontak Discord. Login admin dan login pelanggan **terpisah total** (cookie, sesi, dan rute berbeda).
Halaman publik: landing dengan tutorial, `/product`, dan `/legal` (Terms of Service, Privacy Policy, Contact Support).

**Tema:** gradasi hitam ke magenta `#FF00FF` dengan kaca cair ala iOS; wallpaper dan logo opsional (`docs/PANDUAN-TAMPILAN.md`).

## 8. Alur kerja di VS Code (empat terminal)

Buka folder proyek di VS Code, lalu **Terminal -> Run Task...**. Task dikelompokkan ke **4 grup terminal** supaya rapi:

| Grup terminal | Isi |
|---|---|
| 1. Backend | `ISID: Backend` (server), `ISID: Stop Server` |
| 2. Tunnel | `ISID: Tunnel` (Cloudflare) |
| 3. Test / Migrate | `Run Tests`, `Run Migration`, `Check Server`, `Check Remote Endpoint`, `Check Database Integrity`, `Create Admin`, `Seed Example Products`, `Deploy Netlify`, `Regenerate API docs`, dan `Logs` (log langsung) |
| 4. Backup | `Backup Database`, `Restore Database`, `Backup .env (encrypted)` |

Daftar lengkap:

| Task | Fungsi |
|---|---|
| `ISID: Start All` | Membuka Backend, Tunnel, dan Logs sekaligus |
| `ISID: Backend` / `Tunnel` / `Logs` | Satu per satu |
| `ISID: Stop Server`, `Check Server`, `Check Remote Endpoint` | Kontrol dan pemeriksaan |
| `ISID: Run Tests`, `Run Migration`, `Check Database Integrity` | Pengembangan |
| `ISID: Backup Database`, `Restore Database`, `Backup .env (encrypted)` | Pengamanan data |
| `ISID: Create Admin`, `Seed Example Products`, `Deploy Netlify`, `Regenerate API docs` | Pengaturan |

Panel F5 (Run and Debug) punya *ISID Backend (debug)*, *ISID Tests: file yang sedang dibuka*, dan *ISID CLI script (pilih)*.
**Semua script mendukung `--help`** yang hanya mencetak bantuan dan **tidak** menjalankan apa pun (ini sengaja, agar mengetik `--help` tidak memicu backup/restore sungguhan).

## 9. Perintah CLI (juga tersedia sebagai `.bat`)

| `.bat` | Script | Catatan |
|---|---|---|
| `SETUP.bat` | `init-env.js` + `migrate.js` | Membuat `.env` (secret acak), menyiapkan database. Aman diulang. |
| `CREATE-ADMIN.bat` | `create-admin.js` | Hanya satu admin. Password **tampil** saat diketik di CMD lokal (permintaan pemilik). |
| `RESET-ADMIN-PASSWORD.bat` | `reset-admin-password.js` | Mencabut semua sesi admin dan membuka kunci login. |
| `RESET-ADMIN-2FA.bat` | `reset-admin-2fa.js` | Mematikan 2FA bila HP hilang. |
| **`START-ISID.bat`** | `ops.js` + `START-BACKEND.bat` | **Satu klik** harian: backend + memastikan service Cloudflared RUNNING + tes hostname. Idempoten (tidak membuat dobel). `START-ISID.bat nopause` untuk Task Scheduler. |
| **`STOP-ISID.bat`** | `stop-server.js` | Menghentikan backend dengan rapi; service Cloudflared tetap hidup kecuali Anda pilih (`STOP-ISID.bat service`). |
| **`STATUS-ISID.bat`** | `ops.js status` | Hanya membaca: backend, service, hostname, jalur publik. |
| `SETUP-TUNNEL.bat` | `setup-tunnel.js` | Mengisi `ISID_BACKEND_ORIGIN` di `.env` (hostname permanen). Tidak membaca Tunnel Token. |
| `START-BACKEND.bat` | `index.js` | Backend saja (dipanggil `START-ISID.bat`); memulai ulang otomatis bila dashboard/restore meminta (kode keluar 75); menolak backend ganda. |
| `START-TUNNEL.bat` | - | Hanya untuk mode lama (tanpa service); bawaan: tunnel dijalankan service Windows `Cloudflared`. |
| `STOP-SERVER.bat` | `stop-server.js` | Versi lama `STOP-ISID.bat` (hanya backend). |
| `BACKUP-DATABASE.bat` | `backup.js` | `VACUUM INTO` + manifest + salinan storage, diverifikasi. |
| `RESTORE-DATABASE.bat` | `restore.js` | Hanya saat backend berhenti; wajib mengetik `RESTORE`. |
| `CHECK-DATABASE.bat` | `check-database.js` | Integrity, foreign key, migrasi, SHA-256 semua file Lua. Tidak mengubah apa pun. |
| `BACKUP-ENV.bat` / `RESTORE-ENV.bat` | `backup-env.js` / `restore-env.js` | `.env` terenkripsi (scrypt + AES-GCM) dengan passphrase Anda. |
| `DB-MIGRATE.bat` | `migrate.js` | Backup pengaman otomatis bila database sudah berisi data. |
| `SEED-PRODUCTS.bat` | `seed-products.js` | 3 contoh produk dari `seed/` (opsional). |
| `DEPLOY-NETLIFY.bat` | `deploy-netlify.js` | Tidak pernah macet diam-diam (lihat `docs/AUDIT-V1.md`). |
| `TEST-SERVER.bat` | `check-server.js`, `check-remote.js` | Memeriksa backend lokal dan jalur publik. |

Perintah log: `npm run logs` (atau task "ISID: Logs") mengikuti log secara langsung. Log sudah disaring: password, token, dan secret tidak pernah tertulis.

## 10. Database

SQLite (`data/isid.db`), mode WAL, foreign key aktif, 26 tabel (termasuk `schema_migrations` dan `admins_legacy`, tabel admin V1 yang disimpan, tidak dihapus) dan 8 migrasi. Rincian ada di `docs/DATABASE.md`.
PostgreSQL **tidak didukung** (V1 hanya punya adapter yang belum teruji; dihapus agar tidak menyesatkan).
Migrasi hanya maju: file lama **tidak pernah diedit** (dijaga checksum). Perubahan skema = file migrasi baru, dengan backup `pre-migration` otomatis.

## 11. Keamanan (ringkas; lengkap di `docs/SECURITY.md`)

HTTPS (Netlify + Cloudflare), gateway secret backend-Netlify, otorisasi di server setiap permintaan, validasi input,
rate limit per IP dan per akun, password admin di-hash (Argon2id bila ada, kalau tidak scrypt), 2FA TOTP opsional dengan perlindungan kode ganda,
kunci login 15 menit setelah 5 kali salah, sesi bisa dicabut, token hanya disimpan sebagai hash, CSRF (token + cek Origin), CSP ketat tanpa inline script,
HSTS, `nosniff`, `frame-ancestors none`, perlindungan path traversal, audit log, log tersaring, tidak ada secret di frontend, `.env` tidak ikut backup biasa.
Setiap payload yang dikirim diberi watermark pelacak dan dicatat di `payload_deliveries`. Password pelanggan di-hash (scrypt/Argon2id), akun terkunci 15 menit setelah 8 kali salah, URL webhook Discord disimpan terenkripsi, dan log eksekusi dihapus otomatis setelah 90 hari (dapat diatur).

**Catatan jujur tentang obfuscation:** kode apa pun yang dikirim ke klien bisa diambil pengguna yang sudah berlisensi (misalnya dengan hook `loadstring`).
Sistem ini membatasi **siapa** yang menerima script (per akun, per produk, bisa dicabut, ada watermark), bukan **apa yang bisa dilakukan penerima** setelah script berjalan.

## 12. Test yang benar-benar dijalankan

`npm test` (atau task "ISID: Run Tests") menjalankan **332 test**. Hasil pada pohon proyek final: **332 lulus, 0 gagal, 0 dilewati** (Linux, Node 22; interpreter Lua 5.4 tersedia).
Interpreter Lua sungguhan dipakai untuk menjalankan bootstrap (dengan mock Roblox), dan Chromium sungguhan dipakai untuk uji tampilan pelanggan dan admin saat pengembangan (skrip uji tampilan tidak ikut paket). Rincian per berkas: `docs/LAPORAN-PENGUJIAN.md`.

* Skenario **A-G**: pemilik satu produk tidak bisa membuka produk lain; order dua produk; lisensi kedaluwarsa satu produk; user di-ban; produk baru dari dashboard langsung bekerja; browser mendapat HTML; executor mendapat bootstrap dan payload hanya setelah autentikasi.
* **Akun & login**: password sementara sekali tampil dan wajib ganti, lockout, reset mencabut sesi, urutan pemeriksaan loader (username pemain dulu), consent ToS per versi, tanpa pendaftaran mandiri.
* **Key trial** dan **Discord webhook** (terhadap server Discord tiruan): batas klaim, URL terenkripsi, toggle per tujuan, OWNER vs PUBLIC, pengiriman ulang, kegagalan Discord tidak mengganggu aksi utama.
* **Frontend statis**: sintaks semua JS, versi hukum server = klien, ToS 8 bagian, kamus Indonesia, tanpa sisa alur lama.
* Katalog & versi: draft/published, immutability oleh trigger, rollback, SHA-256, upload, arsip.
* Order & pembayaran: approve paralel (satu lisensi), reject/cancel/refund, bukti pembayaran, notifikasi, isolasi antar pelanggan, Discord di setiap halaman.
* Operasional: backup/restore nyata, backup tahan `fsync` ditolak (EPERM Windows), restore saat startup, endpoint kontrol lokal, migrasi dari database lama, alat `ops.js`/`setup-tunnel.js`, pemeriksaan statis `.bat`.
* **Script CLI nyata** pada salinan proyek sementara, termasuk alur interaktif di **pty**.
* Keamanan: gateway secret, CSRF, rate limit, lockout, TOTP (vektor RFC 6238), path traversal, log tanpa rahasia, frontend tanpa secret.
* Gateway Netlify (api-proxy dan loader) terhadap backend sungguhan, termasuk saat backend mati.

## 13. Yang BELUM/tidak bisa saya buktikan (baca ini)

* **File `.bat` (termasuk `START-ISID/STOP-ISID/STATUS-ISID`), task VS Code, deploy Netlify sungguhan, service Cloudflared/Cloudflare Tunnel, dan CDN Netlify belum pernah dijalankan di Windows/akun Anda.** Logikanya ada di script Node yang diuji; pembungkus `.bat` hanya memanggilnya. Ikuti `docs/UJI-MANUAL.md` setelah pemasangan.
* **Roblox dipalsukan (mock) di test.** Pencarian username/UserId (admin membuat akun) dan `Players.LocalPlayer` sungguhan belum diuji dari sini. Bila PC Anda tidak bisa menjangkau API Roblox, UserId bisa diketik manual. Roblox dapat mengubah API-nya.
* **Roblox OAuth tidak diimplementasikan.** Tidak ada klaim sebaliknya.
* **Executor sungguhan dan Luau asli belum diuji** (Delta, dll.). Lapisan kompatibilitas HTTP (`request`, `http_request`, `syn.request`, `http.request`, `fluxus.request`), GUI login, penyimpanan device key/key trial lokal, `identifyexecutor`, dan `gethwid` hanya diuji dengan mock di Lua 5.4.
* **Discord asli belum diuji** (webhook diuji terhadap server Discord tiruan). Gunakan tombol **Send test**.
* **Teks Terms of Service/Privacy adalah draf**; teks ToS asli Anda tidak dapat dipulihkan kata demi kata. Periksa `legal-content.js`. Bukan nasihat hukum.
* **Data klien bisa dipalsukan** (username/executor/device/HWID yang dilaporkan loader).
* **Wallpaper dan logo GIF Anda tidak ikut paket** dan belum dilihat di tema.
* **Shader dan Drone Show hanyalah placeholder.** Camera Studio memakai script asli dari kit lama Anda (file `seed/*.lua` tidak diubah dari kit asal).
* Upload Lua dari dashboard lewat Netlify dibatasi sekitar 4 MB (batas body Function).
* SQLite di satu PC cocok untuk skala kecil-menengah. Untuk trafik besar, pindah ke VPS.
* Verifikasi pembayaran **manual**; tidak ada webhook gateway (tabel `webhook_events` disiapkan untuk masa depan, belum dipakai).

## 14. Masalah umum

| Gejala | Penyebab / solusi |
|---|---|
| Website tampil, tetapi login/produk "Server maintenance" | PC mati, backend atau service Cloudflared tidak jalan. Klik `STATUS-ISID.bat` lalu `START-ISID.bat`. |
| `GATEWAY_NOT_CONFIGURED` | `ISID_BACKEND_ORIGIN` / `ISID_GATEWAY_SECRET` belum terpasang di Netlify. Jalankan `DEPLOY-NETLIFY.bat`. |
| 403 "Access denied" dari backend | `ISID_GATEWAY_SECRET` di `.env` berbeda dengan di Netlify. Jalankan ulang `DEPLOY-NETLIFY.bat`. |
| Error CSRF/Origin | `ISID_PUBLIC_URL` di `.env` tidak sama dengan alamat Netlify. Perbaiki lalu restart backend. |
| `Hostname tunnel: PERINGATAN` | Route di Cloudflare belum dibuat / salah arah. Harus menuju `http://127.0.0.1:8787`; lihat `PANDUAN-INSTALASI-AWAL.md` langkah 4. |
| Node.js terlalu lama | Pasang Node.js LTS terbaru (22.13+). |
| Pelanggan: "Username atau password salah" | Cek ejaan; admin **Reset password**. Akun terkunci 15 menit setelah 8 kali salah (admin bisa **Unlock**). |
| Loader: "username tidak sama" | Pemain yang menjalankan script bukan pemilik akun ISID itu. Lihat **Executions**. |
| Pelanggan lama tidak bisa login setelah upgrade | Normal: belum punya password. Admin -> Users -> **Reset password**. |
| Lupa password admin | `RESET-ADMIN-PASSWORD.bat`. HP 2FA hilang: `RESET-ADMIN-2FA.bat`. |
| Database rusak / salah hapus | `CHECK-DATABASE.bat`, lalu `RESTORE-DATABASE.bat` dari backup terbaru yang lolos verifikasi. |
