# Audit proyek ISID Hub V1

Tanggal audit: November 2026. Objek: `ISID-Hub-v1.0_1.zip` (V1) yang Anda kirim.

## Metode dan batasan (baca dulu)

* Audit dilakukan dengan **membaca kode V1 dari ZIP**, menjalankan 31 test bawaannya, lalu membandingkannya dengan spesifikasi V2 Anda.
  Setiap temuan di bawah memuat **bukti** (file dan baris di V1) yang bisa Anda cek sendiri di ZIP V1.
* Akar masalah deploy Netlify disimpulkan dari **kode `.bat` V1** dan **disimulasikan dengan stub** CLI buatan sendiri (`server/tests/fixtures/netlify-stub.js`),
  **bukan** dengan Netlify CLI asli dan bukan di Windows Anda. Perilaku persis CLI asli (apa yang ia tanyakan) tidak bisa saya buktikan dari sini;
  perbaikan V2 dirancang agar benar **apa pun** yang ditanyakan CLI (lihat temuan 1).
  Pengecualian: satu log nyata dari Windows Anda (Netlify CLI 27.16.2) memperlihatkan bug di langkah [1/4]; itu sudah diperbaiki dan ditambahkan sebagai test (lihat bagian bug di bawah). Langkah [2/4] sampai [4/4] masih belum pernah dijalankan dengan CLI asli.
* Audit ini bukan audit keamanan pihak ketiga. Lihat `docs/SECURITY.md` untuk kontrol yang ada dan batasannya.

## Yang sudah baik di V1 dan dipertahankan

| Hal | Keterangan |
|---|---|
| Entitlement per produk | `UNIQUE(user_id, product_id)` dan kunci asing komposit `(license_id, user_id, product_id)` di `001_init.sql` |
| Tanpa paket npm | Hanya modul bawaan Node.js; tidak ada rantai pasokan pihak ketiga |
| Token disimpan sebagai hash, CSRF, gateway secret, CSP tanpa inline script, rate limit, TOTP | Dipertahankan dan diuji ulang |
| Loader dua respons (browser vs executor) | Sudah ada di V1; V2 menambah kasus 404/maintenance/disabled dan `Vary`/`no-store` yang diuji |
| 31 test otomatis | Dipertahankan. 27 tidak berubah; 4 disesuaikan dengan spesifikasi V2 (tes peran admin diganti tes "tidak ada route multi-admin"; "approve ganda ditolak" menjadi "approve idempoten"; tes maintenance memeriksa Lua pemberitahuan berisi kontak Discord; tes gateway ditambah pemeriksaan cache). Total sekarang 135 test |

## Temuan dan perbaikan di V2

| # | Temuan | Bukti di V1 | Dampak | Perbaikan di V2 | Pembuktian |
|---|---|---|---|---|---|
| 1 | **Deploy Netlify "macet" di `[3/4] Memasang environment variable di Netlify...`** | `DEPLOY-NETLIFY.bat` baris 57-61: `call netlify env:set ... >nul` dan `... --secret >nul 2>nul`. Keluaran (termasuk pertanyaan/prompt CLI) dibuang ke `nul`, sementara stdin masih tersambung ke konsol | Bila CLI meminta sesuatu (login browser, konfirmasi menimpa variabel, pilihan scope untuk secret), pertanyaan **tidak terlihat** dan CLI menunggu selamanya: layar tampak diam | Logika dipindah ke `server/scripts/deploy-netlify.js`: keluaran selalu tampil, stdin ditutup (mode non-interaktif), tiap langkah punya **batas waktu** dan detak progres, flag eksplisit (`--force --site --context --scope`), deteksi belum-login walau `status` exit 0, fallback tanpa `--secret`, dan **petunjuk manual lengkap** bila otomatis gagal | `deploy.test.js` (20 test): meniru perilaku V1 (stub menunggu jawaban), lalu membuktikan V2 tidak pernah menggantung dan selalu menjelaskan. **Stub; lihat bug nyata di bawah untuk satu keluaran CLI asli** |
| 2 | Banyak admin dengan peran owner/admin/support | `services/admins.js` baris 122-139, `routes/admin/auth.js` baris 45 | Bertentangan dengan spesifikasi V2 (administrator tunggal); permukaan serangan lebih luas | Satu admin. Migrasi `002`: admin tambahan **dipindah** ke `admins_legacy` (tidak dihapus, tidak bisa login), indeks unik `idx_admins_single` di database, pembuatan admin kedua ditolak | `v2-cli.test.js`, "V2 admin tunggal" |
| 3 | Prompt password di CMD disembunyikan | `scripts/prompt.js` baris 17-21 (`setRawMode`) | Bertentangan dengan permintaan Anda (password tampil di CMD/terminal lokal, tersembunyi hanya di dashboard); sulit menempel | `prompt.js` memakai `readline` biasa, tidak menyembunyikan ketikan. Dashboard web tetap menyembunyikan password | `v2-cli.test.js`: alur `create-admin` di **terminal sungguhan (pty)**, password terlihat di layar |
| 4 | Status order tidak sesuai spesifikasi (`approved`, tanpa `refunded`) | `001_init.sql` baris 89 | Tidak bisa merepresentasikan refund; istilah tidak sesuai | Migrasi `003`: `pending/paid/rejected/cancelled/refunded`; data `approved` menjadi `paid`; `subtotal/discount/paid_at/refunded_at` | `v2-orders.test.js`, uji migrasi V1 -> V2 |
| 5 | Versi yang sudah published masih bisa diubah | Tidak ada trigger/CHECK di `001_init.sql` | Pelanggan yang mengandalkan versi tertentu bisa menerima isi berbeda; integritas rilis lemah | `release_status` draft/published + trigger immutability di database (migrasi `006`) + SHA-256 per file | `v2-catalog.test.js` (mencoba `UPDATE`/`DELETE` langsung ke SQLite: ditolak) |
| 6 | Tidak ada tabel riwayat/pengiriman/bukti/notifikasi | `001_init.sql` | Tidak ada jejak perubahan lisensi dan pembayaran, tidak ada bukti bayar | Migrasi `005`: `license_history`, `payload_deliveries`, `payment_attachments`, `payment_history`, `notifications`, `webhook_events` | `v2-orders.test.js` |
| 7 | Backup minim, tanpa restore | `scripts/backup.js`: salin DB + `private/`, **tanpa verifikasi**, tanpa manifest, tanpa skrip restore; retensi menghapus backup tertua setelah 30 tanpa membedakan backup **manual** | Backup yang rusak baru ketahuan saat dibutuhkan; backup manual bisa hilang; tidak ada jalur pemulihan | `VACUUM INTO` + manifest + verifikasi `integrity_check` + salinan storage + salinan luar; retensi **tidak pernah menghapus backup manual**; restore wajib konfirmasi `RESTORE` dan membuat backup pengaman `pre-restore`; `.env` tidak ikut tetapi punya brankas terenkripsi | `v2-ops.test.js` (restore nyata, restore saat startup), `v2-cli.test.js` |
| 8 | Adapter PostgreSQL tidak pernah diuji | `src/db/index.js` baris 75-104 | Klaim dukungan yang tidak terbukti | Dihapus dari V2; SQLite saja (sesuai spesifikasi) | - |
| 9 | Tidak ada cara menghentikan/merestart backend dengan rapi, tidak ada pembacaan log langsung, monitor server minim | `scripts/` V1 tidak punya `stop`/`restore`/`check-database`/`tail-logs` | Operasi harian bergantung menutup jendela; pemulihan sulit | `STOP-SERVER.bat` (endpoint kontrol lokal, hanya loopback + token turunan), restart kode 75, `tail-logs`, Server Monitor jujur ("unavailable" bila tidak terbaca), log tersaring | `v2-ops.test.js`, `v2-system.test.js` |
| 10 | Tidak ada kontak pembayaran Discord | - | Alur pembayaran manual tidak punya instruksi di antarmuka | `system_settings` (`discord_username`, `discord_invite_url`, instruksi) + tampil di landing, produk, dashboard, checkout, dukungan, footer, error; kontak cadangan di Netlify Function saat backend mati | `v2-orders.test.js`, uji UI Chromium |
| 11 | Skrip CLI V1 menjalankan pekerjaan nyata walau hanya diberi `--help` | mis. `scripts/backup.js` tidak punya pemeriksaan argumen apa pun | Mengetik `--help` bisa memicu backup/aksi sungguhan | Setiap script CLI memanggil `lib/help.js` paling awal: `--help` hanya mencetak bantuan | `v2-cli.test.js` ("--help di SEMUA script aman") |
| 12 | Dashboard admin V1 | Menu: Overview, Products, Users, Licenses, Orders, Server, Security | Belum mencakup Lua Storage, Database, Audit Logs, Settings, Orders & Payments dengan bukti bayar | Dashboard V2 dengan 11 tampilan, data nyata, aksi berbahaya minta konfirmasi password | Uji UI Chromium (alur admin dan pelanggan), `v2-system.test.js` |

## Bug yang ditemukan saat pengembangan V2 (dan sudah diperbaiki)

Ini bukan bug V1, tetapi dicatat agar jujur bahwa test dan pemakaian nyata menemukan masalah. Baris bertanda "Bug nyata" ditemukan dari log Windows pengguna, bukan dari test, sehingga membuktikan bahwa test dengan CLI tiruan saja tidak cukup:

| Bug | Ditemukan oleh | Perbaikan |
|---|---|---|
| Mengganti payload **draft** menghilangkan isi lama (tidak ada salinan) | `v2-catalog.test.js` | `copyToTrash`: isi lama disalin ke `storage/trash/` sebelum ditimpa |
| Logger menghasilkan JSON tidak valid dan **melempar error** bila meta berisi kunci sensitif (penyaringan regex pada string JSON) | `v2-system.test.js` | Penyaringan berbasis objek (`redactMeta`), kunci seperti `password`/`token`/`cookie` bernilai `[DISEMBUNYIKAN]` |
| `backup.js` tanpa database membuat database kosong lalu mem-backup-nya | `v2-cli.test.js` | Backup hanya berjalan bila file database sudah ada |
| Gambar bukti bayar diblokir CSP (`blob:`) | Uji UI Chromium | `img-src` menambah `blob:` dan `https:` (di backend dan `netlify.toml`) |
| Overflow horizontal di layar 390px, tombol logout hilang di mobile | Uji UI Chromium (tangkapan layar) | Perbaikan CSS grid/`min-width`, tombol akun tetap tampil di mobile |
| `[object Object]` pada blok instruksi Discord di halaman order | Uji UI Chromium | Memakai objek kontak dengan benar |
| **Bug nyata dari log Windows pengguna:** `DEPLOY-NETLIFY.bat` berhenti di `[1/4] Login: login belum berhasil` padahal sudah login (Netlify CLI 27.16.2, Node 24.13.1). `netlify status` pada akun yang sudah login tetapi folder belum di-link mencetak "Current Netlify User ..." lalu "Error: You don't appear to be in a folder that is linked to a project" dan **berhenti dengan kode non-nol**; skrip menganggap kode non-nol berarti belum login | Log asli yang ditempel pengguna (bukan test) | Login dinilai dari isi keluaran (`scripts/lib/netlify-login.js`); "belum login" hanya bila keluaran menyatakannya; bila tidak jelas, skrip mencoba `netlify login` (yang menjawab "Already logged in") dan lanjut. Peringatan Node 24 DEP0190 (`spawn` dengan array argumen + `shell:true`) dihilangkan lewat `spawnPlan` di `scripts/lib/proc.js` |
| Test "tanpa nama bank hard-coded" kadang gagal (±5% per run): pencocokan nama bank (BCA, BRI, dst.) mencocokkan potongan acak di UUID heksadesimal | Menjalankan tiap suite terpisah | Test hanya memeriksa teks yang terbaca manusia, dengan batas kata |

## Yang tidak diaudit / di luar jangkauan

* Infrastruktur Netlify, Cloudflare, dan Windows Anda (lihat `docs/UJI-MANUAL.md`).
* Perilaku Roblox Users API sungguhan dan executor sungguhan.
* Penetration test pihak ketiga.
