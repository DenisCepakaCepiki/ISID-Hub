# Prosedur uji manual (HARUS Anda jalankan sendiri)

Test otomatis (332, lihat `docs/LAPORAN-PENGUJIAN.md`) berjalan di Linux dengan Roblox, Netlify, dan Cloudflare yang **ditiru atau tidak ada**.
Bagian di bawah adalah hal-hal yang **tidak bisa dibuktikan dari lingkungan pembuat** dan karena itu belum boleh dianggap berfungsi sebelum Anda mencobanya.

Cara memakai: kerjakan berurutan. Catat hasil di tabel bagian 12. Bila sebuah uji gagal, **berhenti**, salin pesan error persisnya (tanpa secret), dan baru lanjut setelah diperbaiki.
Ganti `SITE` dengan alamat Netlify Anda (mis. `isekaistationid.netlify.app`), `API` dengan alamat tunnel (mis. `api.domainanda.com`), dan `SLUG` dengan slug produk.
Perintah `curl.exe` tersedia di Windows 10/11 (ketik `curl.exe`, **bukan** `curl` di PowerShell).

---

## 1. Windows dan file `.bat`

| # | Uji | Langkah | Hasil yang diharapkan |
|---|---|---|---|
| 1.1 | Node.js | `_cek-node.bat` lewat `SETUP.bat` | `[OK] Node.js v22.x` (atau lebih baru). Bila Node lama: pesan jelas, tidak lanjut |
| 1.2 | Setup | Klik dua kali `SETUP.bat` | `.env` terbentuk; migrasi sukses; jendela **tidak menutup** sebelum Anda menekan tombol |
| 1.3 | Setup ulang | Jalankan `SETUP.bat` lagi | "Semua secret sudah terisi (tidak diubah)" |
| 1.4 | Buat admin | `CREATE-ADMIN.bat`, ketik username dan password | **Password tampil saat diketik**; "Administrator ... berhasil dibuat". Jalankan lagi: "sudah ada ... hanya mendukung SATU" |
| 1.5 | Satu klik | Pastikan service Cloudflared terpasang (bagian 3), lalu `START-ISID.bat` | RINGKASAN: `Backend lokal: OK`, `Cloudflared Service: OK (RUNNING)`, `Hostname tunnel: OK`. Jendela BACKEND (minimized) tetap terbuka |
| 1.6 | Idempoten | Klik `START-ISID.bat` **lagi** | Tidak membuka backend kedua; "sudah berjalan"; tidak ada tunnel/service ganda |
| 1.7 | Status | `STATUS-ISID.bat` | Menampilkan backend, service, hostname, jalur publik; tidak mengubah apa pun |
| 1.7a | Stop | `STOP-ISID.bat` | Backend berhenti rapi; service Cloudflared **tetap RUNNING** (jawab N atau biarkan 10 dtk). Lalu `START-ISID.bat` menyalakan lagi |
| 1.7b | Restart otomatis | Dashboard -> Server Monitor -> Restart (password) | Jendela backend menulis "meminta dijalankan ulang ... 2 detik" lalu ONLINE lagi |
| 1.8 | `--help` aman | Di Command Prompt (folder proyek): `node server\scripts\backup.js --help` | Hanya teks bantuan; **tidak ada** backup baru di `backups\` |
| 1.9 | Reset admin | `RESET-ADMIN-PASSWORD.bat` (saat login di dashboard) | Password baru berlaku; sesi dashboard lama keluar sendiri |
| 1.9a | Backup tidak EPERM | Tunggu backup otomatis atau `BACKUP-DATABASE.bat` | Backup selesai tanpa `EPERM: operation not permitted, fsync` (bug nyata dari log Anda yang sudah diperbaiki) |
| 1.10 | Spasi/OneDrive | Jalankan dari folder yang namanya mengandung spasi (mis. `C:\Isid Hub`) | Semua `.bat` tetap jalan. **Jika gagal, laporkan:** path berspasi adalah risiko `.bat` yang tidak bisa saya uji. (Disarankan tetap memakai `C:\ISID-Hub`) |
| 1.11 | Otomatis saat login (opsional) | Task Scheduler: `START-ISID.bat nopause`, restart PC | Setelah login, `STATUS-ISID.bat` semua OK. Belum teruji |

## 2. VS Code

| # | Uji | Langkah | Hasil |
|---|---|---|---|
| 2.1 | Buka folder | VS Code -> Open Folder, percayai folder | Tidak ada error pada `.vscode/*.json` |
| 2.2 | Start All | Terminal -> Run Task -> `ISID: Start All` | Backend, Tunnel, Logs berjalan di panel terminal terpisah (task Tunnel hanya memeriksa status service Cloudflared; untuk menyalakannya pakai `START-ISID.bat`) |
| 2.3 | Grup terminal | Jalankan `ISID: Run Tests` dan `ISID: Backup Database` | Test di grup 3 (Test/Migrate), backup di grup 4 (Backup) |
| 2.4 | Debug | F5 -> "ISID Backend (debug)" (saat backend lain mati) | Backend start; breakpoint bisa dipasang |
| 2.5 | Run Tests | `ISID: Run Tests` | Seluruh test lulus. Test Lua dan test terminal pty **dilewati otomatis** bila tidak ada interpreter Lua/curl atau python3 (normal di Windows). **Suite ini baru pernah dijalankan di Linux**; bila ada test yang gagal khusus di Windows, laporkan nama test dan pesannya |

## 3. Cloudflare Tunnel (service Windows `Cloudflared`)

| # | Uji | Langkah | Hasil |
|---|---|---|---|
| 3.1 | Service terpasang | Command Prompt: `sc query Cloudflared` | `STATE: RUNNING`. Dashboard Cloudflare: tunnel **HEALTHY** |
| 3.2 | Route | Dashboard -> tunnel -> Published application routes | Ada 1 route: hostname Anda -> `HTTP` `127.0.0.1:8787` |
| 3.3 | Hostname menjangkau backend | `STATUS-ISID.bat` / langkah [4/4] `START-ISID.bat` | `Hostname tunnel: OK` |
| 3.4 | Backend terlindungi | `curl.exe -i https://API/api/v1/status` | **HTTP 403** "Access denied" (tanpa gateway secret) |
| 3.5 | Dengan secret (hanya uji; jangan simpan/tempel di chat) | `curl.exe -i -H "X-ISID-Gateway-Secret: NILAI_DARI_.env" https://API/api/v1/status` | **HTTP 200**, JSON `"ok":true` |
| 3.6 | Alamat tetap | Restart PC; `START-ISID.bat` | `API` yang sama kembali bekerja tanpa deploy ulang Netlify |
| 3.7 | Token aman | Cari token tunnel di folder proyek (Explorer: cari `eyJ` pada `.env`, `logs`) | **Tidak ada** (ISID Hub tidak pernah menyimpannya) |

## 4. Netlify

| # | Uji | Langkah | Hasil |
|---|---|---|---|
| 4.1 | Dry run | `node server\scripts\deploy-netlify.js --dry-run` | Hanya mencetak rencana; tidak ada perintah dijalankan |
| 4.2 | Deploy | `DEPLOY-NETLIFY.bat` | Ringkasan `[OK]` per langkah. **Pada langkah [3/4]: keluaran Netlify CLI terlihat; jendela tidak diam lebih dari ~60 detik** tanpa pesan |
| 4.3 | **Uji bug V1** | Saat langkah [3/4], perhatikan apakah ada permintaan (login/konfirmasi) | Permintaan **tampil di layar**. Bila CLI menunggu tanpa keluaran, skrip memberi tahu setelah batas waktu dan menampilkan **cara manual** (tekan Enter setelah memasang dua variabel di dashboard Netlify) |
| 4.4 | Variabel terpasang | Dashboard Netlify -> Environment variables | `ISID_BACKEND_ORIGIN` dan `ISID_GATEWAY_SECRET` ada; secret tidak bisa dibaca ulang |
| 4.5 | Status publik | `curl.exe -i https://SITE/api/v1/status` | 200, `"ok":true` |
| 4.6 | `TEST-SERVER.bat` | Klik dua kali | Semua `[OK]` untuk lokal **dan** publik |
| 4.6a | Seed tidak terbuka | Browser: `https://SITE/seed/isid-camera-studio.lua` dan `https://SITE/server/src/app.js` | **404** keduanya (hanya `site/public` yang dipasang). Bila isi file tampil: ada deploy lama yang salah; hapus di dashboard Netlify |
| 4.6b | Halaman hukum | `https://SITE/legal` | Tampil Terms of Service (8 bagian), Privacy, Contact Support |
| 4.7 | Header keamanan | `curl.exe -I https://SITE/` | Ada `Strict-Transport-Security`, `X-Frame-Options: DENY`, `Content-Security-Policy` |

**Bila 4.2/4.3 masih macet di CLI asli**, laporkan: versi `netlify --version`, dan teks terakhir yang tampil. Skrip dirancang agar benar apa pun yang CLI tanyakan, tetapi saya **tidak bisa memverifikasi CLI asli** dari lingkungan pembuat (hanya CLI tiruan).

## 5. `/loader/SLUG`: browser vs executor

Buat satu produk berstatus Active (atau `SEED-PRODUCTS.bat`).

| # | Uji | Perintah / langkah | Hasil |
|---|---|---|---|
| 5.1 | Browser | Buka `https://SITE/loader/SLUG` di browser | **Halaman HTML produk** (bukan kode Lua) |
| 5.2 | Executor (curl) | `curl.exe -i https://SITE/loader/SLUG` | HTTP 200, `Content-Type: text/plain`, isi **kode Lua** (bukan HTML), `Cache-Control: no-store`, header `Vary` |
| 5.3 | Browser dipaksa lewat curl | `curl.exe -i -H "Sec-Fetch-Dest: document" -H "Sec-Fetch-Mode: navigate" https://SITE/loader/SLUG` | HTML |
| 5.4 | Slug tidak ada | Browser: `/loader/tidak-ada` | Halaman 404 |
| 5.5 | Slug tidak ada (executor) | `curl.exe -i https://SITE/loader/tidak-ada` | HTTP 200 berisi Lua pemberitahuan (bukan error mentah) |
| 5.6 | Maintenance | Aktifkan Maintenance di Server Monitor; ulangi 5.1 dan 5.2 | Browser: halaman maintenance (503). Executor: Lua pemberitahuan **dengan kontak Discord** |
| 5.7 | Backend mati | `STOP-SERVER.bat`; ulangi 5.1 dan 5.2 | Browser: maintenance 503. Executor: Lua "Server maintenance" + Discord cadangan |
| 5.8 | Tidak ada payload bocor | `curl.exe https://SITE/loader/SLUG` (tanpa login) lalu cari isi script produk | Isi script produk **tidak** ada di respons; hanya bootstrap |

## 6. Akun, login, dan persetujuan (Roblox asli untuk pencarian username)

**Gunakan akun Roblox uji. Sistem ini tidak pernah meminta password atau cookie Roblox; jika ada halaman yang meminta, hentikan.**

| # | Uji | Langkah | Hasil |
|---|---|---|---|
| 6.1 | Buat akun | Admin -> Users -> Create account (username Roblox uji, password kosong) | Akun dibuat; **password sementara tampil sekali** (tutup lalu buka lagi: tidak bisa dilihat). Bila "Roblox user not found" padahal username benar: PC tidak menjangkau Roblox; isi UserId manual |
| 6.2 | Pop-up persetujuan | Buka `/dashboard` (browser baru/privat) | Pop-up Terms/Privacy **sebelum** login; tidak bisa ditutup tanpa memilih; tidak ada tombol daftar; tertulis "Belum punya akun? Hubungi atau masuk Discord" |
| 6.3 | Login + ganti password | Login dengan password sementara | Pop-up setuju lagi, lalu **wajib ganti password**; setelah itu dashboard terbuka |
| 6.4 | Password salah | Salah ketik 8 kali | Pesan sama untuk username tak dikenal dan password salah; akun terkunci 15 menit; admin bisa **Unlock** |
| 6.5 | Reset | Admin -> Reset password | Password lama langsung tidak berlaku; sesi pelanggan keluar |
| 6.6 | Bahasa | Tombol ID/EN | Seluruh halaman berganti; pilihan bertahan setelah muat ulang |
| 6.7 | Webhook akun | Lihat channel Discord OWNER | Pesan "akun dibuat"/"reset" **tanpa password** (kecuali opsi Anda nyalakan) dengan `@Anda` |

## 7. Executor sungguhan

Butuh executor Roblox yang Anda percaya (gunakan di akun uji). Hasil tiap executor bisa berbeda: **catat nama dan versinya.**

| # | Uji | Langkah | Hasil |
|---|---|---|---|
| 7.1 | Bootstrap | Jalankan `loadstring(game:HttpGet("https://SITE/loader/SLUG"))()` | Muncul GUI **ISID SECURITY LOGIN** dengan tab MASUK dan KEY TRIAL; password tampil bertopeng; tombol LIHAT/TUTUP bekerja |
| 7.2 | Belum punya lisensi | Login dengan akun uji tanpa lisensi | Pesan produk belum dibeli + kontak Discord; **tidak ada** script produk dijalankan |
| 7.3 | Dengan lisensi | Beri lisensi lewat Licenses -> Grant, jalankan lagi | Script produk berjalan |
| 7.4 | Perangkat diingat | Jalankan lagi di executor yang sama | Tidak perlu mengetik ulang (device key tersimpan di file lokal executor; bila executor tidak mendukung file, harus login tiap kali) |
| 7.4a | Username berbeda | Login dengan akun ISID milik orang lain **dari akun Roblox yang berbeda** | Ditolak `USERNAME_MISMATCH` **sebelum** password/lisensi diperiksa; tercatat di Admin -> Executions (denied) |
| 7.4b | Password sementara | Pakai akun yang belum mengganti password sementara | Ditolak dengan petunjuk mengganti password di website |
| 7.4c | Log eksekusi | Admin -> Executions | Baris untuk tiap percobaan: username Roblox, executor, device id, IP, map/game; **cocokkan dengan kenyataan** (HWID/nama executor bisa kosong bila executor tidak menyediakannya) |
| 7.4d | Webhook OWNER | Channel Discord OWNER | Pesan pelacakan per eksekusi (username, device, IP, map); `@Anda` bila mention aktif |
| 7.4e | Key trial | Admin -> Trial Keys -> buat key; di loader tab KEY TRIAL, tempel key | Script berjalan; klaim tampak di tabel; setelah masa trial habis ditolak dengan pesan beli; akun Roblox yang sama tidak bisa klaim ulang |
| 7.5 | Cabut | Suspend/Revoke lisensi di dashboard; jalankan lagi | Ditolak ("License ..."), tanpa script |
| 7.6 | API HTTP tidak didukung | Executor tanpa `request`/`http_request`/`syn.request`/`http.request` | Pesan "Unsupported executor HTTP API" |
| 7.7 | Script Camera Studio | Jalankan produk dari `SEED-PRODUCTS.bat` | Script berjalan seperti di kit lama (isi file tidak diubah dari V1) |
| 7.8 | Nama executor | Jalankan bootstrap di tiap executor yang Anda punya, lalu buka Settings -> Executors -> tabel "Executors seen" | Nama yang dilaporkan executor tampil **persis**. Catat (mis. Delta mungkin melapor "Delta" atau "Delta X"). Bila nama tidak ada di daftar, klik **Allow** |
| 7.9 | Daftar izin | Policy `enforce`; jalankan dari executor **di daftar** lalu dari yang **di luar** daftar | Yang di daftar: script berjalan. Yang di luar: pesan "Executor ... is not supported" + Discord, script tidak berjalan |
| 7.10 | Executor tanpa identifyexecutor | Jalankan di executor yang tidak punya `identifyexecutor` | Pesan "Your executor could not be identified" (di `enforce`). Bila pelanggan sah terkena ini, pakai `monitor` atau minta mereka ganti executor |
| 7.11 | Policy monitor | Ubah ke `monitor`; jalankan dari executor di luar daftar | Script berjalan (tidak diblokir). Percobaan itu tetap tercatat; di tabel, executor tsb tampil "Not allowed" (belum ada di daftar). Kembalikan ke `enforce` setelah selesai |

## 8. Isolasi produk (prioritas utama)

Siapkan **dua produk** (A dan B) dan **dua akun uji** (X dan Y).

| # | Uji | Hasil |
|---|---|---|
| 8.1 | Beri X lisensi A saja; X menjalankan loader A | Berjalan |
| 8.2 | X menjalankan loader **B** | Ditolak ("Product not purchased"); **tidak ada** script B |
| 8.3 | Y (tanpa lisensi) menjalankan A | Ditolak |
| 8.4 | Dashboard X (`/dashboard`) | A tampil lengkap; B tampil sebagai belum dimiliki, tanpa loader dan tanpa changelog |
| 8.5 | Cabut lisensi A milik X; lisensi B milik X (bila ada) | Hanya A berhenti; B tetap berjalan |

## 9. Pembayaran manual lewat Discord

| # | Uji | Hasil |
|---|---|---|
| 9.1 | Pelanggan membuat order | Status `pending`; instruksi dan kontak Discord (`denis.cepaka`, tautan server) tampil di halaman order, produk, dashboard, footer |
| 9.2 | Tombol Discord | *Join ISID Discord* membuka `https://discord.gg/isekai-station-id-703563738367655937`; *Copy Discord Username* menyalin `denis.cepaka` |
| 9.3 | Unggah bukti | Hanya PNG/JPEG/WebP; berkas lain ditolak |
| 9.4 | Approve | Admin Approve -> status `paid`, lisensi hanya untuk produk di order |
| 9.5 | Approve dua kali (dua tab) | Hanya satu lisensi |
| 9.6 | Refund | Order `refunded`; lisensi dicabut bila `refund_revokes_license` = 1 |
| 9.7 | Ubah kontak | Settings -> ubah username Discord; tampil di semua halaman setelah muat ulang |
| 9.8 | Webhook PUBLIC | Approve order | Pengumuman "user telah membeli" di channel PUBLIC hanya berisi username Roblox (tanpa IP/HWID/mention). Coba sembunyikan nama per akun (Users -> Account) |
| 9.9 | Send test | Discord Webhooks -> tiap kartu **Send test** | Embed uji muncul di channel yang benar. Saklar *Enabled* dimatikan -> tidak ada kiriman |

## 10. Backup dan restore (latihan)

Ikuti `docs/BACKUP-RESTORE.md` bagian 7 (buat backup, ubah data, restore, periksa). **Ini satu-satunya cara tahu backup Anda berguna.**
Tambahan: cabut flashdisk salinan luar lalu buat backup: CLI harus melaporkan "Salinan luar: GAGAL" dengan alasan (bukan diam).

## 11. Keamanan cepat

| # | Uji | Hasil |
|---|---|---|
| 11.1 | Login admin salah 5 kali | Terkunci 15 menit (pesan jelas) |
| 11.2 | 2FA | Aktifkan di Security; login minta kode; kode yang sama tidak bisa dipakai dua kali |
| 11.3 | Admin kedua | Tidak ada cara membuat (UI tidak menyediakan; `CREATE-ADMIN.bat` menolak) |
| 11.4 | Sumber halaman | Menu browser "View source" pada `/admin`: tidak ada secret, tidak ada `<script>` inline |
| 11.5 | Konsol browser | Buka DevTools -> Console pada `/`, `/dashboard`, `/admin`: tidak ada error CSP merah |
| 11.6 | Akses Lua langsung | `https://SITE/storage/private/SLUG/v1.0.0.lua` dan `https://API/storage/...` | 404/403, **bukan** isi Lua |
| 11.7 | Log | Buka `logs\isid-*.log` setelah login: tidak ada password/token |

## 12. Lembar hasil

| Bagian | Lulus? | Tanggal | Catatan (versi executor, pesan error persis) |
|---|---|---|---|
| 1 Windows/.bat | | | |
| 2 VS Code | | | |
| 3 Cloudflare | | | |
| 4 Netlify | | | |
| 5 Loader | | | |
| 6 Akun & login | | | |
| 7 Executor | | | |
| 8 Isolasi produk | | | |
| 9 Pembayaran & webhook Discord | | | |
| 10 Backup/restore | | | |
| 11 Keamanan | | | |

**Jangan menjual lisensi ke pelanggan sungguhan sebelum bagian 3, 4, 5, 6, 7, 8, dan 10 lulus.**
