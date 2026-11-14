# Panduan setelah terpasang: pemakaian harian ISID Hub v1.0.0

Untuk Anda yang **sudah** memasang (lihat `PANDUAN-INSTALASI-AWAL.md`). Intinya: **instalasi tidak diulang**. Cloudflare sudah berupa Windows Service, `.env` dan database sudah ada. Setiap hari cukup tiga file ini.

## 1. Rutinitas harian

| Kapan | Klik dua kali | Apa yang terjadi |
|---|---|---|
| PC baru menyala / sebelum jualan | **`START-ISID.bat`** | Memeriksa konfigurasi, menyalakan backend (jendela terpisah yang di-minimize, **jangan ditutup**), memastikan service Cloudflared RUNNING, menguji jalur hostname. **Aman diklik berulang kali**: bila backend/tunnel sudah hidup, tidak dibuat yang kedua |
| Ingin tahu kondisi sekarang | **`STATUS-ISID.bat`** | Hanya membaca: backend, service Cloudflared, hostname tunnel, jalur publik Netlify. Tidak mengubah apa pun |
| Mau mematikan / PC dipakai lain | **`STOP-ISID.bat`** | Menghentikan backend dengan rapi. Service Cloudflared bawaannya **tetap hidup** (tombol N otomatis dalam 10 dtk); pilih Y hanya bila ingin memutus tunnel juga |

Bila jendela backend tidak sengaja tertutup: klik `START-ISID.bat` lagi. Tidak ada data yang hilang.
Selama PC mati/backend mati: website tampil, tetapi login dan script menampilkan "Server maintenance" beserta kontak Discord Anda.

### Jalankan otomatis saat Windows login (opsional, belum teruji di Windows)
Service Cloudflared sudah otomatis hidup bersama Windows. Untuk backend: **Task Scheduler** -> *Create Basic Task* -> Trigger *When I log on* -> Action *Start a program*:
Program `C:\ISID-Hub\START-ISID.bat`, Argument `nopause`, *Start in* `C:\ISID-Hub`. Uji dengan Restart PC, lalu periksa `STATUS-ISID.bat`. Bila gagal, kembali ke klik manual.

## 2. Yang JANGAN diulang

* `SETUP.bat`: aman (tidak mengubah isi `.env`), tetapi tidak perlu.
* `CREATE-ADMIN.bat`: hanya satu admin; akan ditolak. Lupa password: `RESET-ADMIN-PASSWORD.bat`.
* Pemasangan service Cloudflared (`cloudflared service install ...`): sudah terpasang. Memasang dua kali menimbulkan error.
* `DEPLOY-NETLIFY.bat`: **hanya** bila file website (`site\`) berubah (wallpaper/logo, teks hukum, tampilan) atau hostname/secret berubah. **Mengganti script produk, membuat akun, memberi lisensi, mengatur webhook TIDAK memerlukan deploy**: semuanya lewat dashboard admin.

## 3. Pekerjaan admin sehari-hari (di `/admin`)

| Tugas | Cara |
|---|---|
| Pelanggan baru membeli | **Users -> Create account** (username Roblox; password kosong = dibuat sementara dan tampil **sekali**; salin dan kirim lewat Discord). Lalu **Licenses -> Grant** produknya. Pelanggan login di website dengan username Roblox + password itu dan **wajib menggantinya** |
| Pelanggan lupa password | **Users** -> pilih pelanggan -> tab **Account** -> **Reset password**: password sementara baru (sekali tampil), password lama langsung tidak berlaku, semua sesi dicabut |
| Akun terkunci (8 kali salah = 15 menit) | Tab **Account** -> **Unlock** |
| Memperbarui script produk | **Products** -> produk -> kartu **Update the script (new version)**: pilih/tempel file `.lua`, isi nomor versi dan changelog, *Publish* dan *Make current* sudah tercentang. Pelanggan yang menjalankan loader berikutnya langsung memakai versi baru |
| Versi baru bermasalah | Produk yang sama -> tombol **Roll back to vX** (kembali ke versi published sebelumnya). Versi yang sudah published tidak bisa diubah isinya, hanya diganti versi baru atau di-rollback |
| Key trial gratis | **Trial Keys** -> buat (pilih produk, lama trial, batas klaim, masa berlaku). Bagikan `ISID-XXXX-XXXX-XXXX`; semua klaim tampak di tabel. Bisa dicabut/dipulihkan |
| Memantau eksekusi | **Executions**: setiap percobaan login loader (berhasil maupun ditolak) lengkap dengan username Roblox, executor, device, IP, map/game. Log disimpan 90 hari (bisa diubah di Settings), lalu dihapus otomatis |
| Notifikasi Discord | **Discord Webhooks**: kartu OWNER (privat, menyebut `@Anda`) dan PUBLIC (pengumuman "user telah membeli", hanya username Roblox). Tiap kartu punya URL, tombol **Send test**, dan saklar *Enabled* + daftar event masing-masing |
| Mode perawatan | **Server Monitor -> Maintenance mode** |

## 4. Backup (sekali setup, lalu otomatis)

* **Otomatis**: `ISID_AUTO_BACKUP=true` (bawaan); interval diatur di Settings. Isi `ISID_BACKUP_EXTERNAL_PATH` di `.env` (flashdisk/OneDrive) agar salinannya tidak hanya di satu disk.
* **Manual**: `BACKUP-DATABASE.bat` (database + file Lua privat, diverifikasi) dan `BACKUP-ENV.bat` (`.env` terenkripsi dengan passphrase Anda; tanpa `.env` backup database tidak bisa membuka 2FA admin).
* **Pulihkan**: `STOP-ISID.bat` -> `RESTORE-DATABASE.bat` (ketik `RESTORE`) -> `CHECK-DATABASE.bat` -> `START-ISID.bat`. **Latih sekali** sebelum benar-benar butuh (`docs\BACKUP-RESTORE.md`).

## 5. Upgrade dari pemasangan lama (V2.0 / V2.1) tanpa kehilangan data

Folder lama **tidak ditimpa**. Pelanggan lama **tidak punya password** setelah upgrade (sebelumnya memakai frasa profil), jadi Anda perlu membuatkannya (langkah 9).

1. `STOP-ISID.bat`. Lalu **salin seluruh folder lama** ke flashdisk (jaring pengaman di luar program).
2. Jalankan `BACKUP-DATABASE.bat` dan `BACKUP-ENV.bat` di folder lama.
3. Ekstrak `ISID-Hub-v1.0.0.zip` ke folder **baru** (mis. `C:\ISID-Hub-v1`).
4. Salin dari folder lama ke folder baru: file **`.env`**, folder **`data\`**, **`storage\`** (isi `private\<slug>\*.lua`), **`backups\`**.
5. Di folder baru jalankan `SETUP.bat` (menambah pengaturan baru, **tidak mengubah** secret yang ada), lalu `DB-MIGRATE.bat`: migrasi `008` ditambahkan; backup `pre-migration` dibuat otomatis lebih dulu. Hanya **menambah** tabel/kolom, tidak ada data dihapus.
6. `CHECK-DATABASE.bat`: harus berakhir "database SEHAT" dan semua Lua "cocok dengan SHA-256".
7. `START-ISID.bat` (pakai service Cloudflared yang sama; **tidak perlu** memasangnya ulang).
8. `DEPLOY-NETLIFY.bat` (memasang halaman baru: login password, ToS, dua bahasa).
9. Di `/admin` -> **Users**: untuk **setiap pelanggan lama** pilih **Reset password** lalu kirim password sementaranya. Admin Anda sendiri tidak berubah.
10. Pengaturan lama `allow_registration` / `verification_mode` tidak dipakai lagi (dibiarkan di database, tidak berbahaya).
11. Setelah semuanya normal beberapa hari, folder lama boleh diarsipkan. Jangan hapus sebelum Anda yakin.

## 6. Mengganti Gateway Secret (disarankan bila pernah terlihat orang lain / dikirim lewat chat)

1. `STOP-ISID.bat`.
2. Buat nilai baru: buka Command Prompt di folder ISID Hub, ketik `node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"` dan salin hasilnya.
3. Buka `.env` dengan Notepad, ganti **hanya** nilai setelah `ISID_GATEWAY_SECRET=`, simpan. (Jangan menempel nilai ini ke chat.)
4. `START-ISID.bat`, lalu `DEPLOY-NETLIFY.bat` (memasang nilai baru ke Netlify). Selama jeda singkat itu permintaan publik 403; setelah deploy normal kembali.

## 7. Bila ada masalah

| Gejala | Langkah |
|---|---|
| Pelanggan: "Server maintenance" | `STATUS-ISID.bat` -> lihat baris yang `GAGAL`/`PERINGATAN` -> `START-ISID.bat` |
| `Cloudflared Service: GAGAL (tidak RUNNING)` | Izinkan UAC saat diminta; atau buka *Services* Windows -> `Cloudflared` -> Start. Cek status tunnel di dashboard Cloudflare (HEALTHY) |
| Login pelanggan: "Username atau password salah" | Cek ejaan username; **Reset password** dari admin bila perlu |
| Loader menolak: "username tidak sama" | Pemain yang menjalankan script bukan pemilik akun ISID itu: username di Roblox harus **sama** dengan username akun. Lihat **Executions** untuk alasannya |
| Log lengkap | `logs\isid-TANGGAL.log` (tanpa password/token). Atau `npm run logs` |

Detail teknis: `README-ID.md`, `docs\DEPLOYMENT.md`, `docs\SECURITY.md`, `docs\UJI-MANUAL.md`.
