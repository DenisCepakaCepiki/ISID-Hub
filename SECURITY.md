# ISID Hub v1.0.0: Keamanan

Dokumen ini menjelaskan apa yang dilindungi, bagaimana, dan **apa yang tidak** dilindungi. Setiap kontrol yang disebut punya test otomatis kecuali ditandai **(belum teruji)**.
Ini bukan audit pihak ketiga dan tidak menjamin sistem kebal serangan.

## 1. Aset dan ancaman utama

| Aset | Ancaman | Kontrol utama |
|---|---|---|
| Script Lua produk | Dikirim ke yang belum bayar; diambil lewat URL statis | Lua hanya di `storage/private/`, dikirim setelah cek lisensi **per produk**, tidak pernah lewat web statis, tidak pernah ditampilkan di dashboard |
| Akun pelanggan | Tebak password, pemakaian kredensial orang lain, akun bersama | Password di-hash, kunci 15 menit setelah 8 kali salah, pencocokan username pemain, device key, log eksekusi |
| URL webhook Discord | Bocor lewat database/API/log | Disimpan terenkripsi, tidak pernah dikembalikan API, disamarkan di audit/log |
| Akun administrator | Tebak password, pencurian sesi, CSRF | scrypt/Argon2id, kunci login, 2FA TOTP, cookie HttpOnly+SameSite=Strict, CSRF, sesi 8 jam yang bisa dicabut |
| Backend di PC Anda | Diakses langsung dari internet, bypass Netlify | Bind `127.0.0.1`, hanya lewat Cloudflare Tunnel, wajib `X-ISID-Gateway-Secret` |
| Data pelanggan | Kebocoran antar pelanggan | Otorisasi per baris (`user_id`) di server, FK komposit, uji isolasi |
| Secret (`.env`) | Bocor ke frontend, log, backup, GitHub | Tidak pernah di frontend/log/backup biasa; `.gitignore`; brankas terenkripsi |
| Pembayaran | Order palsu, lisensi ganda | Pembayaran manual diverifikasi admin; approve transaksional dan idempoten; jumlah dibayar dicocokkan dengan total |

## 2. Autentikasi administrator (tunggal)

* **Satu administrator.** Dijaga indeks unik di database; CLI/API menolak admin kedua.
* **Password:** minimal 12 karakter, maksimal 128, tidak boleh mengandung username, tidak boleh satu karakter berulang. Hash **Argon2id bila didukung Node, kalau tidak scrypt** (N=2^17, r=8, p=1; parameter tersimpan di hash sehingga bisa dinaikkan nanti). Perbandingan memakai waktu konstan; respons login gagal memiliki waktu seragam (tidak membocorkan username valid).
* **Kunci login:** 5 kali salah -> terkunci 15 menit (HTTP 423). Di atasnya ada rate limit per IP: login admin 10 percobaan / 10 menit.
* **2FA TOTP (RFC 6238)** opsional (sangat disarankan): diuji dengan vektor resmi RFC. Kode yang sudah dipakai **tidak bisa dipakai ulang** (`totp_last_step`). Secret 2FA disimpan terenkripsi (AES-GCM) dengan kunci turunan `ADMIN_SESSION_SECRET`.
* **Sesi:** token acak tinggi-entropi, **hanya hash SHA-256 yang disimpan**; kedaluwarsa 8 jam; bisa dicabut satu per satu atau semua; mengganti password, reset password, dan reset 2FA mencabut semua sesi admin.
* **Aksi berbahaya** (restart server, restore database) meminta **password ulang** (dan kode 2FA bila aktif) serta dibatasi 5 kali / 10 menit. Restore juga wajib mengetik `RESTORE`.
* **Reset darurat hanya dari PC server:** `RESET-ADMIN-PASSWORD.bat` dan `RESET-ADMIN-2FA.bat`. Tidak ada jalur reset lewat internet, sehingga siapa pun yang tidak menguasai PC Anda tidak bisa mereset akun admin. **Konsekuensinya:** siapa pun yang punya akses fisik/akun Windows ke PC itu bisa mereset admin. Lindungi PC Anda (kunci layar, akun Windows ber-password, disk terenkripsi seperti BitLocker).
* Cookie admin (`isid_admin`) dan cookie pelanggan (`isid_user`) **terpisah**; sesi pelanggan tidak bisa memanggil API admin.

## 3. Jalur jaringan dan gateway

```
Internet -> Netlify (CDN + Functions) -> Cloudflare Tunnel -> 127.0.0.1:8787
```

* Backend hanya mendengarkan `127.0.0.1`; tidak ada port yang dibuka di router/firewall Anda.
* **`X-ISID-Gateway-Secret`** wajib (default aktif saat `NODE_ENV=production`). Dibandingkan dengan waktu konstan. Tanpa itu backend menjawab 403, jadi siapa pun yang mengetahui alamat tunnel tidak bisa memanggil API langsung.
* Header `X-ISID-*` dari klien **dibuang** oleh Function; IP asli klien diteruskan lewat `X-ISID-Client-IP` dan **hanya dipercaya bila secret gateway cocok**.
* **Endpoint kontrol lokal** (`/internal/control/*`, dipakai `STOP-SERVER.bat`): hanya dari socket loopback, tanpa header proxy (`X-Forwarded-For`, `CF-Connecting-IP`), dan dengan token turunan `ADMIN_SESSION_SECRET`. Dari jalur publik (Netlify/Cloudflare) ia berpura-pura tidak ada (404). Diuji, termasuk percobaan lewat header proxy.
* **Rotasi `ISID_GATEWAY_SECRET`:** ubah di `.env`, jalankan `DEPLOY-NETLIFY.bat` (memasang nilai baru di Netlify), restart backend. Selama jeda singkat itu permintaan publik akan 403.
* **Rotasi `ADMIN_SESSION_SECRET`:** akan membuat secret 2FA tidak bisa didekripsi. Jalankan `RESET-ADMIN-2FA.bat` **sebelum** mengganti, lalu aktifkan 2FA lagi sesudahnya. Semua sesi lama menjadi tidak valid.

## 4. Pelanggan, executor, dan lisensi

* **Akun pelanggan dibuat admin; tidak ada pendaftaran mandiri.** Login web dan loader memakai **username Roblox + password ISID**. Password minimal 8 karakter, tidak boleh terlalu umum atau mengandung username; di-hash dengan scrypt/Argon2id (sama seperti admin); tidak pernah keluar lewat API, audit, atau log. Password sementara (dibuat acak oleh sistem) tampil **sekali** kepada admin dan **wajib diganti** pada login pertama; selama belum diganti, API lain ditolak (`PASSWORD_CHANGE_REQUIRED`).
* **Kunci akun:** 8 kali salah -> terkunci 15 menit (password benar pun ditolak selama terkunci); admin bisa membuka. Ada pembatas tambahan per username (serangan dari banyak IP tetap dibatasi) dan per IP. Pesan galat **sama** untuk "username tidak ada" dan "password salah" (tidak membocorkan akun). Reset password / ganti password mencabut semua sesi web dan device key akun itu.
* **Loader:** urutan pemeriksaan di server: (1) **username pemain yang menjalankan = username terdaftar** (bila beda: `USERNAME_MISMATCH`), (2) password, (3) UserId terdaftar, (4) password sementara sudah diganti, (5) **lisensi per produk**. Setiap hasil (berhasil/ditolak + alasan) masuk `execution_logs`. Sistem **tidak pernah** meminta password atau cookie Roblox dan tidak memakai Roblox OAuth.
* **Device key** (30 hari, maksimal 3 perangkat, dapat diatur): hanya hash yang disimpan. **Access token** berumur pendek (bawaan 300 detik, dapat diatur di Settings).
* **Persetujuan hukum:** versi Terms of Service/Privacy yang disetujui dan waktunya disimpan per pelanggan; versi yang salah ditolak server. Pop-up persetujuan di klien hanya kenyamanan; pencatatan resminya di server.
* **Key trial:** dibuat hanya admin; format `ISID-XXXX-XXXX-XXXX`; satu akun Roblox satu kali per script; dibatasi jumlah klaim, masa berlaku, dan 3 akun baru per IP; pembatas laju per IP, per akun yang dilaporkan, dan per key; bisa dicabut. Identitas akun Roblox pada trial **dilaporkan klien** (bisa dipalsukan): batasan di atas yang menahan penyalahgunaan, bukan identitas.
* **Per produk:** setiap `POST /payload/request` memeriksa lisensi produk yang diminta. Skenario A-G memeriksa bahwa pemilik produk A ditolak di produk B dengan token yang sama.
* **Lisensi kedaluwarsa** dianggap `expired` saat itu juga, tanpa menunggu pembersih.
* **Rate limit per IP** (per menit kecuali disebut): `auth/login` 20, `web/login` 20, `password/change` 10, `trial` 20, `payload` 30, `loader` 60-120, `orders` 10, `license-check` 60, `proofs` 10 / 10 menit.
* **Kebijakan executor:** bootstrap melaporkan hasil `identifyexecutor()` (atau `getexecutorname()`) ke server pada `POST /api/v1/payload/request`. Pengaturan `executor_policy`: **`enforce`** (bawaan; executor di luar daftar `executor_allowlist` ditolak dengan `EXECUTOR_NOT_ALLOWED` dan script **tidak dikirim**), `monitor` (hanya dicatat), `off`. Daftar bawaan: Delta, Real, Madium; ubah dari Settings. Pencocokan tidak peka huruf besar/kecil ("Delta" juga cocok dengan "Delta X", tidak dengan "Deltaforce"). Klien yang tidak melaporkan apa pun (mis. permintaan HTTP mentah, atau executor tanpa `identifyexecutor`) dianggap di luar daftar. Pemeriksaan ini terjadi **sebelum** lisensi, jadi penolakan tidak membocorkan status lisensi. Setiap percobaan tercatat di `payload_deliveries.executor`; Settings menampilkan tabel "Executors seen". Nilai policy yang rusak dibaca sebagai `enforce`. **Batas:** nama dilaporkan klien (lihat bagian 7).
* **Payload** dikirim dengan `Cache-Control: no-store`, diberi watermark pelacak (jejak HMAC userId+produk+versi), dan tiap pengiriman dicatat di `payload_deliveries` (hasil, alasan, versi, IP ter-hash).
* **Log eksekusi** (`execution_logs`): IP, user agent, executor, device id, `HWID` (hanya hash), place/game/job, username Roblox. Data pribadi pelanggan; dihapus otomatis setelah **90 hari** (Settings -> `execution_log_retention_days`, dipangkas tiap jam). Nilai dari klien disanitasi (karakter kontrol dan panjang dibatasi).
* **Discord webhook:** hanya `https://discord.com/api/webhooks/<id>/<token>` yang diterima (host palsu/SSRF/http ditolak). URL disimpan **terenkripsi AES-256-GCM** (kunci turunan `ADMIN_SESSION_SECRET`), hanya tampil tersamar, tidak masuk audit/log penuh. Karena kuncinya turunan `ADMIN_SESSION_SECRET`, **mengganti secret itu membuat URL webhook tidak bisa didekripsi** (tempel ulang di Admin -> Discord Webhooks). Mention `@everyone`/`@here` dinetralkan. Pengiriman gagal tidak menggagalkan aksi utama. **Password sementara tidak dikirim ke Discord** kecuali opsi *include temporary password* dinyalakan (bawaan mati; hanya webhook OWNER; PUBLIC tidak pernah).

## 5. Web: header, CSRF, XSS

* **CSRF:** semua mutasi ber-cookie wajib membawa `X-CSRF-Token` yang cocok **dan** `Origin` yang diizinkan (`ISID_PUBLIC_URL`). Cookie `SameSite=Strict`, `HttpOnly`, dan `Secure` di production.
* **CSP ketat:** `script-src 'self'` (tanpa inline script dan tanpa `unsafe-inline` untuk script). Frontend dibangun dengan pembangun DOM (`ISID.h`) tanpa `innerHTML`, sehingga data pengguna tidak pernah ditafsirkan sebagai HTML. `img-src` mengizinkan `'self' data: blob: https:` (gambar produk dan pratinjau bukti bayar).
* Header lain: `Strict-Transport-Security`, `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY` (+ `frame-ancestors none`), `Referrer-Policy: no-referrer`, `Cross-Origin-Resource-Policy/Opener-Policy: same-origin`, `Permissions-Policy` (kamera/mikrofon/lokasi/pembayaran dimatikan), `Cache-Control: no-store` pada API.
* **Frontend tidak memegang rahasia:** test memindai semua JS/HTML frontend: tidak ada `ADMIN_SESSION_SECRET`, gateway secret, `document.cookie`, atau penyimpanan token di storage browser.
* **Upload:** Lua dibatasi ukuran (maks 4 MB), ditolak bila mengandung byte NUL; bukti bayar hanya **PNG/JPEG/WebP**, jenisnya ditentukan dari **magic bytes** (bukan nama/Content-Type kiriman klien), maksimal 3 per order, nama file disanitasi (tanpa pemisah path), disimpan privat, hanya pemilik order dan admin yang boleh melihat.
* **Path traversal:** slug, nama backup, dan nama file divalidasi/di-resolve dan dipastikan tetap di dalam folder yang diizinkan (diuji, termasuk nama backup `../../etc/passwd`).
* **Ukuran body:** JSON dibatasi (bawaan 64 KB; unggahan Lua/bukti memakai batas tersendiri). 

## 6. Database, backup, dan log

* Query memakai parameter terikat (tanpa penyambungan string dari input pengguna).
* Integritas rilis dijaga trigger database (lihat `docs/DATABASE.md`): versi published tidak bisa diubah walau seseorang mengedit database langsung.
* **Backup tidak memuat `.env`** (diuji: secret tidak ditemukan di file backup mana pun). `.env` dicadangkan terpisah, terenkripsi (scrypt + AES-256-GCM) dengan passphrase Anda.
* **Log tersaring:** nilai pada kunci bernama persis `password`, `passwd`, `pass`, `secret`, `token`, `totp`, `devicekey`, `accesstoken`, `authorization`, `cookie`, `csrf`, `csrftoken`, atau `passphrase` (tidak peka huruf besar/kecil) diganti `[DISEMBUNYIKAN]` secara rekursif; string yang cocok dengan secret terdaftar dan pola token ISID juga disaring. **Batasan:** kunci dengan nama lain (mis. `gatewaySecret`) tidak otomatis disaring oleh nama kuncinya; kode tidak boleh mencatat nilai rahasia dengan nama kunci sembarang. Test menjalankan alur nyata (login, reset password, gagal login) lalu memastikan password/secret tidak ada di file log maupun keluaran proses.
* IP pada `payload_deliveries` disimpan sebagai HMAC (bukan IP asli); IP mentah hanya ada di sesi, challenge verifikasi, dan audit log.

## 7. Yang BUKAN perlindungan (jujur)

* **Kebijakan executor bukan pembuktian identitas.** Nama executor berasal dari klien. Siapa pun yang memodifikasi `identifyexecutor`, menulis permintaan HTTP sendiri, atau memakai executor yang meniru nama "Delta" bisa lolos. Kontrol ini menyaring executor yang tidak dikenal, klien yang diam, dan pembocor yang ceroboh; ia **tidak** menghentikan pembocor yang sengaja memalsukan nama atau yang meng-hook `loadstring` di executor yang sah. Daftar nama juga ditampilkan dalam pesan penolakan (agar pengguna sah tahu apa yang didukung), jadi penyerang tahu nama apa yang harus dipalsukan. Salah menulis nama executor di daftar akan memblokir pelanggan sah: lihat tabel "Executors seen" dan gunakan `monitor` dulu bila ragu.
* **Obfuscation tidak mencegah pencurian.** Pengguna berlisensi bisa menangkap script saat dieksekusi (mis. hook `loadstring`). Kontrol ada di **siapa** yang menerima (per akun, per produk, bisa dicabut, watermark), bukan **apa** yang bisa dilakukan penerima.
* **Username/UserId/executor/device/HWID yang dilaporkan loader berasal dari klien.** Server mencocokkan username pemain dengan akun yang login dan mencatatnya, tetapi pemain yang menguasai klien (atau executor yang memalsukan `LocalPlayer`) bisa berbohong. Log eksekusi dan webhook adalah **petunjuk**, bukan bukti. Dua orang yang berbagi password dan memakai username yang sama tidak bisa dibedakan selain lewat pembatas perangkat, HWID ter-hash, dan IP.
* **Bukan Roblox OAuth.** Sistem tidak membuktikan bahwa pemain menguasai akun Roblox itu; ia membuktikan bahwa pemain tahu password ISID dan sedang bermain sebagai username yang terdaftar.
* **Satu PC = satu titik kegagalan dan satu titik kepercayaan.** Malware atau akses fisik ke PC Anda berarti seluruh sistem terbuka. Tidak ada HSM atau pemisahan hak.
* **Tidak ada WAF/DDoS khusus** selain yang disediakan Cloudflare/Netlify bawaan akun Anda.
* **Pembayaran manual** bergantung pada ketelitian admin; bukti bayar gambar bisa dipalsukan. Cocokkan dengan mutasi rekening/e-wallet Anda sendiri sebelum menekan Approve.
* **Belum teruji (belum ada pihak ketiga yang menguji):** perilaku di bawah beban tinggi, serangan canggih terhadap Cloudflare/Netlify, dan lingkungan Windows Anda.

## 8. Checklist operasional

1. Pasang **2FA** pada admin (Admin -> Security).
2. Gunakan **Named Tunnel** (bukan Quick Tunnel) dan jangan membuka port di router.
3. **Jangan membagikan `.env`**; jangan mengunggahnya ke GitHub (`.gitignore` sudah mengecualikannya).
4. Aktifkan **BitLocker/enkripsi disk** dan kunci layar PC server.
5. Isi `ISID_BACKUP_EXTERNAL_PATH` (flashdisk/cloud) dan lakukan `BACKUP-ENV.bat` sekali; simpan passphrase di pengelola password.
6. Periksa **Audit Logs** secara berkala; jalankan `CHECK-DATABASE.bat` mingguan.
7. Perbarui Node.js (LTS) dan `cloudflared` secara berkala.
8. Bila URL webhook Discord pernah terlihat orang lain (mis. ditempel di chat), buat webhook baru di Discord dan tempel ulang; bila Gateway Secret pernah terlihat, ganti (`PANDUAN-SETELAH-TERPASANG.md` bagian 6).
9. Cabut akses pelanggan bermasalah segera lewat Licenses (Suspend/Revoke) atau Users (Ban); sesi pelanggan dicabut seketika.

## 9. Pelaporan

Temukan celah? Hubungi admin lewat Discord (`denis.cepaka`). Jangan menyebarkan detail celah sebelum diperbaiki.
