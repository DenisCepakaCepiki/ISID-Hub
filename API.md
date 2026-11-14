# ISID Hub - Referensi API (v1.0.0)

> File ini DIHASILKAN otomatis oleh `npm run docs:api` dari daftar route yang sebenarnya di kode. Jangan diedit manual:
> ubah keterangan di `server/scripts/lib/api-notes.js`, lalu jalankan ulang generator. Test `v2-system.test.js` akan gagal bila dokumen ini
> tidak sama dengan kode.

## Format umum

- Basis URL: website Netlify (`https://<situs>/api/v1/...`), diteruskan Netlify Function ke backend lewat Cloudflare Tunnel. Backend tidak pernah dibuka langsung ke publik.
- Respons JSON sukses: `{ "ok": true, "data": ... }` (daftar berhalaman menambah `"meta": {total,page,pageSize,pages}`).
- Respons JSON gagal: `{ "ok": false, "error": { "code": "KODE", "message": "...", "details": [...] } }`.
- Waktu: epoch milidetik UTC. Harga: bilangan bulat (satuan terkecil mata uang, mis. rupiah).
- Header `X-ISID-Gateway-Secret` dan `X-ISID-Client-IP` hanya dipasang Netlify Function; klien tidak pernah mengirimnya (header `x-isid-*` dari klien dibuang).

## Autentikasi

| Jenis | Cara | Catatan |
|---|---|---|
| admin | cookie HttpOnly `isid_admin` + header `X-CSRF-Token` | Hanya satu administrator. Password di-hash (Argon2id bila didukung, kalau tidak scrypt). 2FA TOTP opsional. |
| pelanggan (web) | cookie HttpOnly `isid_user` + header `X-CSRF-Token` | Login username Roblox + password ISID (akun dibuat admin; password sementara wajib diganti). Tidak pernah meminta password/cookie Roblox. |
| pelanggan (executor) | `Authorization: Bearer <access token>` | Access token berumur pendek (Settings). Device key hanya disimpan sebagai hash SHA-256. |
| lokal | socket loopback + `X-ISID-Control-Token` | Hanya untuk STOP-ISID.bat / STOP-SERVER.bat. |

Mutasi (POST/PUT/PATCH/DELETE) dengan cookie wajib membawa token CSRF yang cocok dan Origin yang diizinkan.

## Kode error yang umum

| HTTP | Kode | Arti |
|---|---|---|
| 400 | VALIDATION_ERROR / BAD_REQUEST / INVALID_JSON | Input tidak valid (`details` menunjuk field). |
| 401 | UNAUTHORIZED / ADMIN_UNAUTHORIZED | Belum login / sesi habis. |
| 403 | CSRF_TOKEN / CSRF_ORIGIN / ACCOUNT_SUSPENDED / PRODUCT_NOT_PURCHASED / LICENSE_EXPIRED / LICENSE_SUSPENDED / LICENSE_REVOKED / LICENSE_PENDING / DEVICE_LIMIT | Ditolak. Kode LICENSE_* dan PRODUCT_NOT_PURCHASED menjelaskan alasan akses produk ditolak. |
| 404 | NOT_FOUND / PRODUCT_UNAVAILABLE / ... | Tidak ada, atau produk tidak tersedia. |
| 409 | CONFLICT / VERSION_PUBLISHED / ORDER_NOT_PENDING / ORDER_PENDING_EXISTS / AMOUNT_MISMATCH / ALREADY_LIFETIME / PRODUCT_IN_USE / USER_HAS_HISTORY / ... | Bertentangan dengan keadaan data saat ini. |
| 413 | PAYLOAD_TOO_LARGE | Body terlalu besar (batas upload Lua dashboard 4 MB). |
| 429 | RATE_LIMITED | Terlalu banyak permintaan (lihat header `Retry-After`). |
| 501 | PROVIDER_NOT_CONFIGURED | Tidak ada payment gateway (pembayaran manual via Discord). |
| 503 | SERVER_MAINTENANCE / PRODUCT_MAINTENANCE / PAYLOAD_MISSING / BACKEND_UNAVAILABLE | Maintenance, atau backend tidak terjangkau (kode terakhir dikirim Netlify Function). |

## Daftar endpoint

`Rate limit` menunjukkan batas per IP (jumlah/jendela detik) yang benar-benar dipasang pada route; `-` berarti tidak ada batas khusus.

### Publik (tanpa login)

| Metode | Path | Autentikasi | Rate limit | Keterangan |
|---|---|---|---|---|
| GET | `/api/v1/health` | tanpa login | - | Cek cepat: backend hidup dan database menjawab. Tanpa data sensitif. |
| GET | `/api/v1/status` | tanpa login | - | Status platform: operational/maintenance/degraded, pesan maintenance, mode login (password), apakah trial aktif, versi dokumen hukum (ToS/Privasi), versi, dan kontak Discord (dari Settings). |
| GET | `/api/v1/contact` | tanpa login | - | Kontak pembayaran/dukungan (username Discord, link undangan, instruksi pembayaran). Sumber tunggal: tabel system_settings. |
| GET | `/api/v1/products` | login opsional | 120/60s | Katalog produk publik (hanya produk live + visibility public). Jika login, menyertakan status kepemilikan pengguna. Query: q, sort=order\|name\|price\|newest. |
| GET | `/api/v1/products/:slug` | login opsional | 120/60s | Detail satu produk (changelog versi published, harga, status). Produk draft/disabled/private/arsip -> 404. Dipakai juga oleh Netlify Function untuk memutuskan halaman produk vs 404. |
| GET | `/loader/:slug` | tanpa login | 60/60s | Satu URL, dua respons: browser -> halaman produk HTML; executor -> Lua bootstrap (text/plain). Selalu Cache-Control: no-store + Vary. Executor selalu HTTP 200 (HttpGet melempar error pada status non-2xx); kondisi tidak normal berupa Lua pemberitahuan dengan header X-ISID-Loader. |
| GET | `/internal/bootstrap/:slug` | tanpa login | 120/60s | Lua bootstrap untuk executor (dipanggil Netlify Function /loader). BUKAN script produk; script produk hanya keluar dari /api/v1/payload/request setelah verifikasi akun + lisensi. |
| POST | `/api/v1/payments/webhook/:provider` | tanpa login | - | Penampung webhook pembayaran. TIDAK ada payment gateway di V2 (pembayaran manual via Discord): tanpa provider terdaftar selalu menjawab 501 PROVIDER_NOT_CONFIGURED, tidak pernah menyetujui order otomatis. |

### Autentikasi pelanggan (executor dan web)

| Metode | Path | Autentikasi | Rate limit | Keterangan |
|---|---|---|---|---|
| POST | `/api/v1/auth/start` | tanpa login | 60/60s | Loader (executor): lanjutkan otomatis dengan device key tanpa mengetik password. Body {productSlug,deviceKey?,robloxUserId,robloxUsername,executor?,hwid?,placeId?,gameId?,jobId?,gameName?,displayName?}. Hasil {status:"verified",accessToken,...} atau {status:"login_required"}. Username pemain dicocokkan dengan akun terdaftar lebih dulu. |
| POST | `/api/v1/auth/login` | tanpa login | 20/60s | Loader (executor): login dengan username Roblox + password ISID (sama dengan dashboard). Body {productSlug,username,password,+info klien}. Urutan: username yang diketik = username pemain? -> password benar? -> UserId terdaftar cocok? -> password sementara sudah diganti? Mengembalikan device key + access token. Salah beruntun mengunci akun sementara. |
| POST | `/api/v1/auth/revoke` | pelanggan (web/executor/device key) | 30/60s | Mencabut sesi/perangkat sendiri: body {sessionId?} atau {all:true}. |
| POST | `/api/v1/auth/web/login` | tanpa login | 20/60s | Login dashboard: body {username,password}. Akun dibuat admin; tidak ada pendaftaran mandiri. Membuat sesi web (cookie HttpOnly isid_user) + token CSRF; mustChangePassword=true jika masih password sementara. |
| POST | `/api/v1/auth/web/logout` | pelanggan (web) | - | Keluar dari sesi web. |
| POST | `/api/v1/auth/password/change` | pelanggan (web) | 10/60s | Ganti password sendiri: body {currentPassword,newPassword}. Mencabut device key/akses executor lama dan sesi web lain; sesi ini dipertahankan. |

### Pelanggan (butuh login)

| Metode | Path | Autentikasi | Rate limit | Keterangan |
|---|---|---|---|---|
| GET | `/api/v1/users/me` | pelanggan (web atau executor) | - | Profil pengguna yang login. |
| GET | `/api/v1/users/me/products` | pelanggan (web atau executor) | - | Produk milik pengguna beserta status lisensi (hanya produk miliknya). |
| GET | `/api/v1/users/me/devices` | pelanggan (web) | - | Daftar perangkat/sesi aktif pengguna (hanya sesi web). |
| GET | `/api/v1/users/me/notifications` | pelanggan (web atau executor) | - | Notifikasi pengguna (order disetujui/ditolak, lisensi akan habis, dll). Query: unread=1. |
| POST | `/api/v1/users/me/notifications/read` | pelanggan (web) | - | Tandai notifikasi dibaca: body {id?} (kosong = semua). |
| GET | `/api/v1/users/me/license-history` | pelanggan (web atau executor) | - | Riwayat perubahan lisensi milik pengguna (license_history). |
| GET | `/api/v1/licenses` | pelanggan (web atau executor) | - | Semua lisensi milik pengguna. |
| GET | `/api/v1/licenses/:productSlug` | pelanggan (web atau executor) | - | Lisensi pengguna untuk satu produk. |
| POST | `/api/v1/licenses/check` | pelanggan (web atau executor) | 60/60s | Cek status lisensi produk: body {productSlug}. Tidak mengirim script. |
| POST | `/api/v1/payload/request` | pelanggan (access token executor) | 30/60s | SATU-SATUNYA jalur pengiriman script Lua produk. Body {productSlug, executor?}; butuh access token. `executor` = nama dari identifyexecutor() (dilaporkan klien, bisa dipalsukan). Urutan pemeriksaan: akun aktif -> maintenance -> produk -> kebijakan executor (EXECUTOR_NOT_ALLOWED bila mode enforce dan di luar daftar izin) -> lisensi produk itu saja -> batas perangkat -> hash SHA-256. Setiap percobaan dicatat di payload_deliveries. |
| POST | `/api/v1/users/me/consent` | pelanggan (web) | 20/60s | Catat persetujuan Terms of Service + Privacy Policy: body {version} (harus sama dengan versi saat ini di /api/v1/status). |
| GET | `/api/v1/orders` | pelanggan (web atau executor) | - | Daftar order milik pengguna. |
| GET | `/api/v1/orders/:id` | pelanggan (web atau executor) | - | Detail order milik pengguna + riwayat pembayaran + bukti bayar + info Discord. |
| POST | `/api/v1/orders` | pelanggan (web) | 10/60s | Buat order (status pending): body {items:[{productSlug}],note?}. Pembayaran manual via Discord. Order pending ganda untuk produk yang sama ditolak (ORDER_PENDING_EXISTS). |
| POST | `/api/v1/orders/:id/cancel` | pelanggan (web) | - | Batalkan order milik sendiri selama masih pending. |
| POST | `/api/v1/orders/:id/proofs` | pelanggan (web) | 10/600s | Unggah bukti bayar (opsional): body {name?,data:base64}. Hanya PNG/JPEG/WebP (dicek dari magic bytes), maksimal 3 per order, ukuran dibatasi Settings (max_proof_bytes). |
| GET | `/api/v1/orders/:id/proofs/:aid` | pelanggan (web) | - | Unduh bukti bayar milik sendiri. |

### Admin: autentikasi dan sesi

| Metode | Path | Autentikasi | Rate limit | Keterangan |
|---|---|---|---|---|
| POST | `/api/v1/admin/auth/login` | tanpa login | 10/600s | Login administrator: body {username,password,totp?}. Rate limit + lockout. Membuat sesi admin (cookie HttpOnly isid_admin) + token CSRF. |
| POST | `/api/v1/admin/auth/logout` | admin (cookie + CSRF) | - | Keluar dari sesi admin. |
| GET | `/api/v1/admin/auth/me` | admin (cookie + CSRF) | - | Profil admin + token CSRF sesi ini. |
| POST | `/api/v1/admin/auth/password` | admin (cookie + CSRF) | 5/600s | Ganti password: body {currentPassword,newPassword}. Sesi lain dicabut. |
| POST | `/api/v1/admin/auth/totp/setup` | admin (cookie + CSRF) | - | Mulai pengaturan 2FA: mengembalikan secret + URI otpauth. |
| POST | `/api/v1/admin/auth/totp/enable` | admin (cookie + CSRF) | 10/600s | Aktifkan 2FA setelah kode pertama valid: body {code} (6 digit). |
| POST | `/api/v1/admin/auth/totp/disable` | admin (cookie + CSRF) | 10/600s | Matikan 2FA: body {password,code?}. |
| GET | `/api/v1/admin/sessions` | admin (cookie + CSRF) | - | Daftar sesi admin aktif. |
| DELETE | `/api/v1/admin/sessions/:id` | admin (cookie + CSRF) | - | Cabut satu sesi admin. |
| POST | `/api/v1/admin/sessions/revoke-others` | admin (cookie + CSRF) | - | Cabut semua sesi admin kecuali yang sedang dipakai. |

### Admin: ringkasan, produk, versi, Lua Storage

| Metode | Path | Autentikasi | Rate limit | Keterangan |
|---|---|---|---|---|
| GET | `/api/v1/admin/overview` | admin (cookie + CSRF) | - | Ringkasan dashboard: total, pendapatan, seri 30 hari, aktivitas terbaru, pemakaian per produk. Semua dihitung dari database. |
| GET | `/api/v1/admin/products` | admin (cookie + CSRF) | - | Daftar produk (admin). Query: search, status, visibility, archived=include\|only, sort, dir. |
| POST | `/api/v1/admin/products` | admin (cookie + CSRF) | - | Buat produk (slug dinamis). Opsional langsung membuat versi pertama: version, lua, fileName, changelog, publish. Tidak ada produk bawaan yang di-hardcode. |
| GET | `/api/v1/admin/products/:id` | admin (cookie + CSRF) | - | Detail produk + versi + statistik. |
| PATCH | `/api/v1/admin/products/:id` | admin (cookie + CSRF) | - | Ubah metadata produk (nama, harga, deskripsi, durasi, visibility, ikon, dll). |
| DELETE | `/api/v1/admin/products/:id` | admin (cookie + CSRF) | - | Hapus produk yang belum dipakai. Ditolak (PRODUCT_IN_USE) bila sudah punya lisensi/order; gunakan arsip. File Lua dipindah ke storage/trash, tidak dihapus permanen. |
| POST | `/api/v1/admin/products/:id/status` | admin (cookie + CSRF) | - | Ubah status: draft \| active \| disabled \| maintenance. Aktif butuh versi published yang utuh. |
| POST | `/api/v1/admin/products/:id/archive` | admin (cookie + CSRF) | - | Arsipkan produk (disembunyikan, data dan lisensi tetap). |
| POST | `/api/v1/admin/products/:id/unarchive` | admin (cookie + CSRF) | - | Kembalikan produk dari arsip. |
| GET | `/api/v1/admin/products/:id/stats` | admin (cookie + CSRF) | - | Statistik produk: lisensi aktif, pengiriman payload 30 hari, ditolak. |
| GET | `/api/v1/admin/products/:id/integrity` | admin (cookie + CSRF) | - | Verifikasi SHA-256 + ukuran semua file Lua produk terhadap database. |
| GET | `/api/v1/admin/products/:id/versions` | admin (cookie + CSRF) | - | Daftar versi (draft/published) + changelog + hash. |
| POST | `/api/v1/admin/products/:id/versions` | admin (cookie + CSRF) | - | Unggah Lua sebagai versi baru: body {version,lua,fileName?,changelog?,publish?,makeCurrent?}. Versi published TIDAK bisa ditimpa. |
| PUT | `/api/v1/admin/products/:id/versions/:vid/payload` | admin (cookie + CSRF) | - | Ganti file Lua pada versi DRAFT. Versi published immutable (dijaga trigger database). |
| PATCH | `/api/v1/admin/products/:id/versions/:vid` | admin (cookie + CSRF) | - | Ubah changelog versi (boleh juga pada versi published). |
| POST | `/api/v1/admin/products/:id/versions/:vid/publish` | admin (cookie + CSRF) | - | Publish versi draft (immutable setelahnya): body {makeCurrent?}. |
| POST | `/api/v1/admin/products/:id/versions/:vid/activate` | admin (cookie + CSRF) | - | Jadikan versi published ini versi aktif. Dipakai juga untuk ROLLBACK ke versi sebelumnya. |
| DELETE | `/api/v1/admin/products/:id/versions/:vid` | admin (cookie + CSRF) | - | Hapus versi DRAFT yang belum pernah dipublish (file dipindah ke trash). |
| GET | `/api/v1/admin/storage` | admin (cookie + CSRF) | - | Inventaris Lua Storage: jumlah file, ukuran, file yatim, file hilang. |
| POST | `/api/v1/admin/storage/verify` | admin (cookie + CSRF) | - | Verifikasi integritas semua file Lua (SHA-256). |

### Admin: pengguna dan lisensi

| Metode | Path | Autentikasi | Rate limit | Keterangan |
|---|---|---|---|---|
| GET | `/api/v1/admin/users` | admin (cookie + CSRF) | - | Daftar pengguna (pencarian, filter status, sortir, halaman). |
| POST | `/api/v1/admin/users` | admin (cookie + CSRF) | - | Daftarkan pengguna manual: body {username,robloxUserId?,email?,notes?}. |
| GET | `/api/v1/admin/users/:id` | admin (cookie + CSRF) | - | Detail pengguna: lisensi, riwayat lisensi, pengiriman payload, perangkat, order. |
| PATCH | `/api/v1/admin/users/:id` | admin (cookie + CSRF) | - | Ubah profil pengguna: body {username?,displayName?,email?,notes?}. |
| DELETE | `/api/v1/admin/users/:id` | admin (cookie + CSRF) | - | Hapus pengguna. Ditolak (USER_HAS_HISTORY) bila punya riwayat; gunakan suspend/ban. |
| POST | `/api/v1/admin/users/:id/status` | admin (cookie + CSRF) | - | Ubah status akun: body {status: active\|suspended\|banned, reason?} (sesi pengguna dicabut). |
| POST | `/api/v1/admin/users/:id/reset-password` | admin (cookie + CSRF) | 30/600s | Atur ulang password pelanggan: body {password?} (kosong = dibuat acak). Password sementara hanya ditampilkan SEKALI; wajib diganti saat login; semua sesi dicabut; akun dibuka kuncinya. |
| POST | `/api/v1/admin/users/:id/unlock` | admin (cookie + CSRF) | - | Buka kunci akun yang terkunci karena terlalu banyak login gagal (tanpa mengganti password). |
| GET | `/api/v1/admin/licenses` | admin (cookie + CSRF) | - | Daftar lisensi (filter produk/status/pengguna). |
| POST | `/api/v1/admin/licenses` | admin (cookie + CSRF) | - | Beri lisensi manual: body {userId,productId,duration?(lifetime\|1d\|7d\|30d\|90d\|custom),customDays?,expiresAt?,purchaseRef?,note?,status?,deviceLimit?}. |
| GET | `/api/v1/admin/license-history` | admin (cookie + CSRF) | - | Riwayat semua perubahan lisensi (license_history). Filter licenseId/userId. |
| GET | `/api/v1/admin/licenses/:id` | admin (cookie + CSRF) | - | Detail lisensi + riwayat. |
| PATCH | `/api/v1/admin/licenses/:id` | admin (cookie + CSRF) | - | Ubah catatan lisensi. |
| POST | `/api/v1/admin/licenses/:id/extend` | admin (cookie + CSRF) | - | Perpanjang lisensi: body {days} (1-3650) atau {lifetime:true}. |
| POST | `/api/v1/admin/licenses/:id/device-limit` | admin (cookie + CSRF) | - | Atur batas perangkat per lisensi: body {limit} (1-20) atau {limit:null} = tanpa batas. |
| POST | `/api/v1/admin/licenses/:id/suspend` | admin (cookie + CSRF) | - | Tangguhkan lisensi (sementara): body {reason?}. |
| POST | `/api/v1/admin/licenses/:id/revoke` | admin (cookie + CSRF) | - | Cabut lisensi: body {reason?}. |
| POST | `/api/v1/admin/licenses/:id/restore` | admin (cookie + CSRF) | - | Pulihkan lisensi yang ditangguhkan/dicabut. |
| DELETE | `/api/v1/admin/licenses/:id` | admin (cookie + CSRF) | - | Hapus lisensi. Hanya yang berstatus pending/revoked/expired (cabut dulu bila aktif); jejak tetap ada di license_history dan audit_logs. |

### Admin: order dan pembayaran

| Metode | Path | Autentikasi | Rate limit | Keterangan |
|---|---|---|---|---|
| GET | `/api/v1/admin/orders` | admin (cookie + CSRF) | - | Daftar order (filter status/produk/pengguna, pencarian, halaman). |
| POST | `/api/v1/admin/orders` | admin (cookie + CSRF) | - | Buat order atas nama pengguna (admin). |
| GET | `/api/v1/admin/orders/:id` | admin (cookie + CSRF) | - | Detail order + item + pembayaran + bukti + riwayat. |
| POST | `/api/v1/admin/orders/:id/approve` | admin (cookie + CSRF) | - | Setujui pembayaran manual: body {paymentRef?,methodLabel?,amount?,acceptMismatch?,note?}. Nominal berbeda dari total order ditolak (AMOUNT_MISMATCH) kecuali acceptMismatch=true. Transaksional dan idempoten (kedua kali -> 200 idempotent:true, tanpa efek ganda). Membuat/memperpanjang lisensi PER produk. |
| POST | `/api/v1/admin/orders/:id/reject` | admin (cookie + CSRF) | - | Tolak order pending: body {note?}. Pengguna diberi notifikasi. |
| POST | `/api/v1/admin/orders/:id/refund` | admin (cookie + CSRF) | - | Refund order paid: body {note?,revokeLicenses?}. Lisensi yang dibuat order ini dicabut (kebijakan refund_revokes_license); lisensi lama yang diperpanjang dilaporkan di manualReview. |
| POST | `/api/v1/admin/orders/:id/discount` | admin (cookie + CSRF) | - | Atur diskon pada order pending: body {discountAmount,note?}. |
| GET | `/api/v1/admin/orders/:id/proofs/:aid` | admin (cookie + CSRF) | - | Unduh bukti bayar order (admin). |
| GET | `/api/v1/admin/payments` | admin (cookie + CSRF) | - | Riwayat pembayaran. Query: status=pending\|confirmed\|failed\|refunded, page. meta.summary berisi total per mata uang dan status. |

### Admin: server, log, audit, pengaturan, database, backup

| Metode | Path | Autentikasi | Rate limit | Keterangan |
|---|---|---|---|---|
| GET | `/api/v1/admin/server` | admin (cookie + CSRF) | - | Server Monitor: uptime, CPU, RAM, disk, status database, tunnel Cloudflare (lokal + jalur remote), metrik request, backup terakhir. Nilai yang tidak bisa dibaca dilaporkan "unavailable", bukan dikarang. |
| POST | `/api/v1/admin/server/maintenance` | admin (cookie + CSRF) | - | Aktif/nonaktifkan mode maintenance global: body {enabled,message?}. |
| POST | `/api/v1/admin/server/restart` | admin (cookie + CSRF) | 5/600s | Restart backend: body {password,totp?} (TOTP wajib bila 2FA aktif). Backend keluar dengan kode 75; START-BACKEND.bat menjalankannya lagi. |
| GET | `/api/v1/admin/logs` | admin (cookie + CSRF) | - | Log server terbaru (secret/token otomatis disamarkan). Query: n. |
| GET | `/api/v1/admin/audit` | admin (cookie + CSRF) | - | Audit log (filter action/entityType/entityId/actorType/search, halaman). |
| GET | `/api/v1/admin/settings` | admin (cookie + CSRF) | - | Semua pengaturan sistem (system_settings). |
| PUT | `/api/v1/admin/settings` | admin (cookie + CSRF) | - | Ubah beberapa pengaturan sekaligus (divalidasi semua dulu; tidak ada perubahan setengah jalan). |
| GET | `/api/v1/admin/database` | admin (cookie + CSRF) | - | Informasi database: ukuran, tabel + jumlah baris, migrasi, WAL, integrity terakhir. |
| POST | `/api/v1/admin/database/integrity` | admin (cookie + CSRF) | - | Jalankan integrity_check + foreign_key_check. |
| POST | `/api/v1/admin/database/optimize` | admin (cookie + CSRF) | - | Optimasi ringan (PRAGMA optimize + ANALYZE + checkpoint pasif). Tidak menghapus data. |
| POST | `/api/v1/admin/database/checkpoint` | admin (cookie + CSRF) | - | WAL checkpoint (TRUNCATE): mengecilkan file WAL. |
| GET | `/api/v1/admin/backups` | admin (cookie + CSRF) | - | Status + daftar backup. |
| POST | `/api/v1/admin/backups` | admin (cookie + CSRF) | 10/600s | Buat backup manual (VACUUM INTO + verifikasi + salinan storage). |
| POST | `/api/v1/admin/backups/:name/verify` | admin (cookie + CSRF) | - | Verifikasi backup (SHA-256 + integrity_check). |
| POST | `/api/v1/admin/backups/:name/restore` | admin (cookie + CSRF) | 5/600s | Jadwalkan restore: body {password,totp?,confirm:"RESTORE",withStorage?}. Dijalankan saat backend dimulai ulang, didahului backup pre-restore. |
| DELETE | `/api/v1/admin/backups/pending-restore` | admin (cookie + CSRF) | - | Batalkan restore yang dijadwalkan sebelum backend dimulai ulang. |
| GET | `/api/v1/admin/webhooks` | admin (cookie + CSRF) | - | Peristiwa webhook yang tercatat (webhook_events). Kosong selama belum ada provider pembayaran otomatis. |

### Internal (hanya dari PC server)

| Metode | Path | Autentikasi | Rate limit | Keterangan |
|---|---|---|---|---|
| POST | `/internal/control/shutdown` | lokal + token kontrol | - | Hentikan backend dengan rapi. HANYA dari PC server: socket loopback + tanpa header proxy + token X-ISID-Control-Token. Selain itu 404. |
| GET | `/internal/control/ping` | lokal + token kontrol | - | Cek kontrol lokal (aturan akses sama dengan shutdown). |

### Lainnya

| Metode | Path | Autentikasi | Rate limit | Keterangan |
|---|---|---|---|---|
| POST | `/api/v1/trial/payload` | tanpa login | 20/60s | Loader (executor), TANPA login: pakai key trial gratis. Body {productSlug,key:"ISID-XXXX-XXXX-XXXX",+info klien}. Klaim pertama memulai jam trial per akun Roblox; panggilan berikutnya melanjutkan selama trial belum habis. Mengembalikan script produk. |
| GET | `/api/v1/admin/executors` | admin (cookie + CSRF) | - | Executor yang melaporkan diri pada percobaan pengiriman payload (?days=1..365, bawaan 30): jumlah percobaan, terkirim, diblokir, pengguna, terakhir terlihat, dan apakah ada di daftar izin saat ini. Nama dilaporkan klien (bisa dipalsukan). |
| GET | `/api/v1/admin/executions` | admin (cookie + CSRF) | - | Log eksekusi loader: username Roblox, UserId, executor, device, HWID (hash), map (PlaceId/GameId/JobId/nama), IP, hasil + alasan. Filter: result, event, search, product, userId, from, to. |
| GET | `/api/v1/admin/executions/summary` | admin (cookie + CSRF) | - | Ringkasan eksekusi N jam terakhir: berhasil/ditolak, per produk, per executor, akun teratas. |
| GET | `/api/v1/admin/trial-keys` | admin (cookie + CSRF) | - | Daftar key trial gratis (ISID-XXXX-XXXX-XXXX) beserta jumlah klaim. Filter: status, productId, search. |
| POST | `/api/v1/admin/trial-keys` | admin (cookie + CSRF) | 30/600s | Buat key trial: body {productId,claimMinutes,maxClaims?,validUntil?,note?,count?}. Key dibuat HANYA oleh admin dan terlihat di dashboard admin. |
| GET | `/api/v1/admin/trial-keys/:id` | admin (cookie + CSRF) | - | Detail key trial + daftar akun yang sudah mengklaim. |
| POST | `/api/v1/admin/trial-keys/:id/revoke` | admin (cookie + CSRF) | - | Cabut key trial (tidak bisa dipakai lagi). |
| POST | `/api/v1/admin/trial-keys/:id/restore` | admin (cookie + CSRF) | - | Aktifkan kembali key trial yang dicabut. |
| DELETE | `/api/v1/admin/trial-keys/:id` | admin (cookie + CSRF) | - | Hapus key trial beserta catatan klaimnya. |
| GET | `/api/v1/admin/discord/webhooks` | admin (cookie + CSRF) | - | Daftar tujuan webhook Discord (OWNER privat dan PUBLIC), katalog event, dan opsi. URL hanya tampil tersamar. |
| POST | `/api/v1/admin/discord/webhooks` | admin (cookie + CSRF) | 300/600s | Tambah tujuan webhook kustom: body {name,scope:"owner"\|"public",url?,events?,enabled?,mentionOwner?}. URL dienkripsi di database. |
| PATCH | `/api/v1/admin/discord/webhooks/:id` | admin (cookie + CSRF) | 300/600s | Ubah tujuan webhook: nama, URL, event yang dikirim, aktif/nonaktif, tag owner. |
| DELETE | `/api/v1/admin/discord/webhooks/:id` | admin (cookie + CSRF) | - | Hapus tujuan webhook kustom (slot bawaan tidak bisa dihapus, hanya dikosongkan). |
| POST | `/api/v1/admin/discord/webhooks/:id/test` | admin (cookie + CSRF) | 20/600s | Kirim pesan uji ke webhook dan tampilkan hasilnya. |
| POST | `/api/v1/admin/discord/announce` | admin (cookie + CSRF) | 10/600s | Kirim pengumuman manual ke semua webhook PUBLIC yang berlangganan: body {title,message}. |

_Total: 124 endpoint._
