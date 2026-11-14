# ISID Hub v1.0.0: Deployment dan operasional

Langkah pertama kali dari nol (pemula): `PANDUAN-INSTALASI-AWAL.md`. Pemakaian harian dan upgrade: `PANDUAN-SETELAH-TERPASANG.md`. Dokumen ini adalah rujukan teknisnya.

```
Roblox executor / browser -> Netlify (statis + Functions) -> Cloudflare Tunnel (service Windows `Cloudflared`) -> Node.js 127.0.0.1:8787 -> SQLite
```

> **Jujur soal pengujian:** Semua yang ada di sini dikembangkan dan diuji di Linux. Netlify, Cloudflare, Windows, dan file `.bat` **belum pernah dijalankan oleh pembuat paket ini**.
> Perintah Cloudflare dan Netlify dicocokkan dengan dokumentasi resmi masing-masing (sumber di akhir). Gunakan `docs/UJI-MANUAL.md` untuk membuktikannya di lingkungan Anda.

## 1. Prasyarat

| Kebutuhan | Keterangan |
|---|---|
| Windows 10/11 yang menyala terus | Backend dan tunnel berjalan di PC ini. Matikan mode Sleep/Hibernate saat colokan listrik terpasang |
| Node.js **22.13+** (LTS) | `node -v`. Tidak ada `npm install` untuk backend |
| Akun Netlify (gratis) | Untuk website statis dan Functions |
| Akun Cloudflare (gratis) + **domain** yang nameserver-nya dikelola Cloudflare | Wajib untuk hostname **permanen** (Published application route). Quick Tunnel tanpa domain hanya untuk uji coba |
| `cloudflared` | `winget install --id Cloudflare.cloudflared`, lalu `cloudflared service install <token>` (perintah persis dari dashboard Cloudflare; **token = password tunnel, jangan dibagikan**) |
| Netlify CLI | Dipasang otomatis oleh `DEPLOY-NETLIFY.bat` (`npm install -g netlify-cli`) |
| VS Code (opsional) | Untuk menjalankan semuanya dari satu jendela |

## 2. Konfigurasi `.env`

`SETUP.bat` membuat `.env` dari `.env.example` dengan dua secret acak (64 hex). Variabel (semua didokumentasikan di dalam `.env.example`):

| Variabel | Fungsi | Catatan |
|---|---|---|
| `PORT` | Port backend | Bawaan 8787; hanya bind `127.0.0.1` (`ISID_HOST` bisa diubah, **tidak disarankan**) |
| `ISID_GATEWAY_SECRET` | Rahasia Netlify <-> backend | Minimal 32 karakter; **sama persis** dengan variabel di Netlify |
| `ADMIN_SESSION_SECRET` | CSRF, 2FA, token kontrol lokal | Minimal 32 karakter. Jangan diganti tanpa membaca `docs/SECURITY.md` bagian 3 |
| `NODE_ENV` | `production` | Mengaktifkan gateway wajib, cookie Secure, HSTS, situs statis tidak disajikan backend |
| `DATABASE_URL` | Kosong = `sqlite:./data/isid.db` | PostgreSQL ditolak |
| `PAYLOAD_STORAGE_PATH` | `./storage` | Lua privat di `<path>/private/` |
| `ISID_BACKUP_DIR`, `ISID_BACKUP_EXTERNAL_PATH`, `ISID_AUTO_BACKUP` | Backup | Lihat `docs/BACKUP-RESTORE.md` |
| `ISID_PUBLIC_URL` | URL website Netlify | Dipakai untuk cek Origin/CSRF dan di bootstrap Lua. **Harus sama** dengan alamat Netlify yang dipakai pengunjung |
| `ISID_BACKEND_ORIGIN` | Hostname permanen tunnel, mis. `https://api.domainanda.com` | Diisi `SETUP-TUNNEL.bat`; dipakai `DEPLOY-NETLIFY.bat`, `START-ISID.bat`/`STATUS-ISID.bat`, dan Server Monitor |
| `ISID_TUNNEL_NAME` | (LEGACY) nama tunnel lokal | Diabaikan bila service `Cloudflared` terpasang |
| `ISID_TUNNEL_METRICS` | `http://127.0.0.1:20241` | Alamat metrics cloudflared (hanya lokal) agar Server Monitor bisa membaca status tunnel |
| `ISID_LOG_RETENTION_DAYS`, `ISID_LOG_REQUESTS` | Log | `ISID_LOG_REQUESTS=all|important|off` |
| `ISID_REQUIRE_GATEWAY`, `ISID_SERVE_SITE`, `ISID_PASSWORD_HASH`, `ISID_COOKIE_SECURE` | Lanjutan/opsional | Bawaan sudah aman untuk production |
| `ISID_DISCORD_USERNAME`, `ISID_DISCORD_INVITE` | (di Netlify) kontak cadangan saat backend mati | Sumber utama kontak ada di dashboard -> Settings |

`.env` **tidak boleh** di-commit atau dibagikan. Cadangkan dengan `BACKUP-ENV.bat`.

## 3. Cloudflare Tunnel sebagai Windows Service (remotely-managed)

Tunnel dikelola dari dashboard Cloudflare (bukan file `config.yml` lokal) dan dijalankan oleh **Windows Service `Cloudflared`** yang menyala otomatis bersama Windows. Langkah klik demi klik: `PANDUAN-INSTALASI-AWAL.md` langkah 4-5. Ringkasnya:

1. Dashboard Cloudflare -> Zero Trust -> Networks -> **Tunnels** -> buat tunnel (Cloudflared) -> pilih Windows -> jalankan perintah `cloudflared.exe service install <token>` di Command Prompt **Administrator**.
2. Tab **Published application routes** -> tambah route: hostname `api.domainanda.com`, Service **HTTP**, URL `127.0.0.1:8787`.
3. `SETUP-TUNNEL.bat` -> isi hostname itu ke `ISID_BACKEND_ORIGIN`.
4. `START-ISID.bat` memastikan service `RUNNING` (meminta UAC bila perlu menyalakannya) dan menguji bahwa hostname menjangkau backend.

* Hostname **tetap** walau tunnel/PC dimulai ulang, jadi Netlify tidak perlu diubah.
* **Tunnel Token tidak pernah dibaca, disimpan, atau diminta oleh ISID Hub** (skrip hanya memeriksa status service lewat `sc`). Jangan menaruhnya di folder proyek atau di chat.
* Periksa dari luar: `https://api.domainanda.com/api/v1/status` harus **403 Access denied** (tidak membawa gateway secret; itu justru tanda backend terlindungi. Jalur yang benar lewat Netlify). `STATUS-ISID.bat` melakukan tes yang benar (dengan secret dari `.env`, tidak dicetak).
* Mode lama (tunnel dari CMD, `START-TUNNEL.bat legacy`) hanya berjalan bila service **tidak** terpasang; tidak disarankan.
* **Quick Tunnel** (`*.trycloudflare.com`, acak dan berubah tiap restart): hanya uji coba; `SETUP-TUNNEL.bat` menolaknya kecuali `--allow-quick`.
* **Belum teruji di akun/Windows Anda**: lihat `docs/UJI-MANUAL.md` bagian 3.

## 4. Netlify

* `netlify.toml`: `publish = site/public`, Functions di `site/netlify/functions` (`api-proxy.mjs` untuk `/api/*`, `loader.mjs` untuk `/loader/<slug>`), pengalihan `/admin`, `/dashboard`, `/product`, dan header keamanan + CSP. **Tidak ada secret di file ini.**
* **Environment variable** (dipasang `DEPLOY-NETLIFY.bat`, atau manual di Site configuration -> Environment variables):
  * `ISID_BACKEND_ORIGIN` = alamat tunnel tanpa `/` di akhir
  * `ISID_GATEWAY_SECRET` = **sama persis** dengan `.env`; dipasang sebagai secret dengan scope `functions` dan `runtime`
  * (opsional) `ISID_DISCORD_USERNAME`, `ISID_DISCORD_INVITE`
* Perubahan variabel **baru berlaku pada deploy berikutnya.**

### Alur `DEPLOY-NETLIFY.bat` / `deploy-netlify.js`
1. Memvalidasi `.env` (secret valid) dan alamat tunnel (`https://host`, tanpa path; menolak karakter yang berbahaya bagi `cmd.exe`).
2. Memeriksa Netlify CLI (menawarkan memasang bila belum ada).
3. **[1/4]** login. Status login dinilai dari **isi keluaran** `netlify status` (bukan dari kode keluarnya): "Current Netlify User" berarti sudah login, walau perintah itu berhenti dengan kode 1 karena folder belum di-link (perilaku Netlify CLI 27.x yang terlihat di log nyata). `netlify login` hanya dijalankan bila benar-benar belum login atau statusnya tidak jelas; `NETLIFY_AUTH_TOKEN` dipakai bila diset.
4. **[2/4]** hubungkan site (`netlify link --name <dari ISID_PUBLIC_URL>`; bila gagal, pilih manual).
5. **[3/4]** `netlify env:set` dengan `--force --site <id>`; secret dengan `--secret --context production deploy-preview branch-deploy --scope functions runtime`. Bila `--secret` gagal/timeout, dicoba tanpa `--secret`. Bila tetap gagal: **petunjuk manual lengkap**, lalu menunggu Anda menekan Enter.
6. **[4/4]** `netlify deploy --prod --dir site/public --functions site/netlify/functions`, lalu memeriksa `/api/v1/status`.
7. **RINGKASAN DEPLOY** `[OK]/[GAGAL]/[CATATAN]` per langkah.

Jaminan desain (diuji dengan CLI tiruan): keluaran tidak pernah disembunyikan; stdin ditutup pada mode non-interaktif; **setiap langkah punya batas waktu** (cek cepat 30 dtk, env 60 dtk, login/link/instal 5 mnt, deploy 10 mnt) dan detak progres; tidak ada langkah yang bisa menggantung diam. Opsi: `--dry-run` (hanya mencetak rencana), `--non-interactive`.
**Teruji sebagian dengan keluaran asli:** langkah [1/4] sudah dicocokkan dengan satu log nyata dari Netlify CLI 27.16.2 / Node 24 di Windows (lihat `docs/AUDIT-V1.md`, bug nyata). **Belum teruji dengan CLI asli:** langkah [2/4], [3/4], dan [4/4] (nama flag mengikuti dokumentasi CLI; bila versi CLI Anda berbeda, jalur manual tersedia). Di Windows, perintah dijalankan lewat shell dengan satu string perintah (bukan array argumen) agar Node 24 tidak mencetak peringatan DEP0190; semua token divalidasi ketat sebelum dijalankan.

## 5. Rutinitas operasional

| Kebutuhan | Cara |
|---|---|
| Menyalakan | **`START-ISID.bat`** (backend + service Cloudflared + tes). Aman diklik berulang kali |
| Status | `STATUS-ISID.bat` (hanya membaca) |
| Mematikan rapi | `STOP-ISID.bat` (backend; service Cloudflared tetap kecuali dipilih); `Ctrl+C` di jendela backend juga aman |
| Restart | Dashboard -> Server Monitor -> Restart (butuh password), `START-BACKEND.bat` menyalakan ulang otomatis (kode keluar 75) |
| Pantau | Dashboard -> Server Monitor: status backend, database, tunnel (dari metrics lokal cloudflared), CPU/RAM/disk, uptime, jumlah request dan error per menit, backup, versi. Yang tidak terbaca ditulis **unavailable** |
| Log | VS Code `ISID: Logs`, atau `npm run logs`; file di `logs/isid-YYYY-MM-DD.log` (dirotasi, dipangkas sesuai `ISID_LOG_RETENTION_DAYS`) |
| Mode perawatan | Dashboard -> Server Monitor -> Maintenance mode. Executor menerima Lua pemberitahuan dengan kontak Discord; browser melihat `/maintenance.html` (503) |
| Periksa dari luar | `TEST-SERVER.bat` (lokal + jalur publik); `node server/scripts/check-remote.js` |
| Update aplikasi | Lihat `PANDUAN-SETELAH-TERPASANG.md` bagian 5 (ekstrak ke folder baru, salin `.env`/`data`/`storage`/`backups`, `SETUP.bat`, `DB-MIGRATE.bat`, `CHECK-DATABASE.bat`, `START-ISID.bat`, `DEPLOY-NETLIFY.bat`) |
| Ganti hostname tunnel | `SETUP-TUNNEL.bat`, lalu `DEPLOY-NETLIFY.bat` |
| Hanya mengubah wallpaper/logo/teks website | `DEPLOY-NETLIFY.bat` (tidak perlu restart backend; teks hukum yang diubah versinya perlu restart backend) |

## 6. Bila backend atau PC mati

Netlify tetap menyajikan halaman statis. `/loader/<slug>` untuk browser menampilkan `/maintenance.html` (503, `Retry-After`); untuk executor mengirim Lua pemberitahuan "Server maintenance" beserta kontak Discord cadangan
(dari variabel Netlify atau nilai bawaan). API (`/api/*`) menjawab 503 JSON. Diuji terhadap backend yang benar-benar dimatikan (`netlify.test.js`). Tidak ada data yang hilang saat PC mati; pelanggan hanya tidak bisa memuat script sampai server menyala lagi.

## 7. Batas kapasitas

* SQLite + satu proses Node di satu PC: cocok untuk skala kecil-menengah (ratusan pelanggan aktif); belum diukur beban nyata di sini.
* Netlify Function membatasi body ~6 MB; unggahan Lua dari dashboard dibatasi 4 MB.
* Throughput dibatasi koneksi internet rumah dan upload ISP Anda. Untuk trafik besar, pindahkan backend ke VPS (salin `data\` dan `storage\`, gunakan Named Tunnel yang sama atau reverse proxy HTTPS).

## 8. Sumber resmi

* Cloudflare: [Create a locally-managed tunnel](https://developers.cloudflare.com/tunnel/features/locally-managed-tunnels/create-local-tunnel/), [Run as a service on Windows](https://developers.cloudflare.com/tunnel/advanced/local-management/as-a-service/windows/)
* Netlify CLI: [env commands](https://cli.netlify.com/commands/env/), [Environment variables overview](https://docs.netlify.com/build/environment-variables/overview/)
