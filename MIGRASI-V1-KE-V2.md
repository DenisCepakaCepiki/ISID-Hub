# Migrasi ISID Hub V1 (lama) -> ISID Hub 1.0.0

> Dokumen ini untuk Anda yang masih memakai **V1 yang paling awal** (admin banyak, tanpa katalog dinamis). Bila Anda sudah memakai V2.0/V2.1, ikuti `PANDUAN-SETELAH-TERPASANG.md` bagian 5 (migrasi `008` ditambahkan otomatis; semua pelanggan perlu **Reset password**). Pada langkah di bawah, "V2" berarti ISID Hub 1.0.0; migrasi `007` dan `008` juga diterapkan otomatis.

Dokumen ini untuk Anda yang **sudah punya data di V1** (pengguna, produk, lisensi, order). Bila Anda memasang dari nol, abaikan dokumen ini.

**Prinsip: tidak ada data yang dihapus.** Perubahan skema dilakukan dengan migrasi baru; admin tambahan dipindah (bukan dibuang) ke tabel arsip;
database dicadangkan otomatis sebelum migrasi pertama.

## Yang berubah (ringkas)

| Hal | V1 | V2 |
|---|---|---|
| Administrator | Banyak, peran owner/admin/support | **Satu** |
| Status order | `pending/approved/rejected/cancelled` | `pending/paid/rejected/cancelled/refunded` |
| Versi produk | Semua bisa diubah | `draft` bisa diubah; `published` **immutable** (dijaga trigger) |
| Pembayaran | Manual tanpa kontak | Manual lewat **Discord** (kontak tampil di seluruh antarmuka) |
| Database | SQLite atau PostgreSQL (belum teruji) | **SQLite saja** |
| Backup | Salin tanpa verifikasi, tanpa restore | Verifikasi, manifest, restore bertahap, retensi aman |
| Password di CMD | Disembunyikan | **Tampil** (permintaan Anda); dashboard web tetap menyembunyikan |
| `--help` di script | Menjalankan pekerjaan nyata | Hanya mencetak bantuan |
| Deploy Netlify | Bisa macet diam-diam | Tidak pernah menggantung; selalu menjelaskan |
| Perilaku API | `POST /admin/orders/:id/approve` kedua kali -> 409 | Idempoten: approve order yang sudah `paid` mengembalikan hasil yang sama tanpa lisensi ganda. Order `rejected/cancelled/refunded` tetap 409. Jumlah dibayar yang berbeda dari total ditolak kecuali `acceptMismatch` |

Rincian lengkap dan alasan: `docs/AUDIT-V1.md`.

## Langkah upgrade (aman)

1. **Hentikan V1**: tutup jendela START-BACKEND dan START-TUNNEL milik V1.
2. **Salin cadangan manual** (di luar program): salin seluruh folder V1 Anda (terutama `.env`, `data\`, `storage\`, `backups\`) ke tempat lain, mis. flashdisk.
   Ini jaring pengaman terakhir; V2 juga membuat backup otomatis, tetapi salinan manual tidak bergantung pada program apa pun.
3. **Ekstrak ZIP V2 ke folder BARU** (mis. `C:\ISID-Hub-V2`). Jangan menimpa folder V1.
4. **Pindahkan data V1 ke folder V2**:
   * salin `.env` dari V1 ke folder V2,
   * salin folder `data\` (berisi `isid.db`) ke folder V2,
   * salin folder `storage\` (berisi `private\<slug>\v<versi>.lua`) ke folder V2.
   (Bila `DATABASE_URL` di `.env` V1 menunjuk PostgreSQL: V2 tidak mendukungnya dan akan menolak start dengan pesan jelas. Anda perlu memindahkan data ke SQLite terlebih dahulu; hubungi saya dengan pesan error persisnya.)
5. **Lengkapi `.env`**: klik dua kali `SETUP.bat`. Ia **tidak mengubah** nilai yang sudah ada (secret tetap sama), hanya menambah pengaturan V2 yang belum ada
   (`ISID_BACKUP_DIR`, `ISID_AUTO_BACKUP`, `ISID_TUNNEL_METRICS`, `ISID_LOG_RETENTION_DAYS`, `ISID_LOG_REQUESTS`, `ISID_BACKUP_EXTERNAL_PATH`) dengan nilai bawaan aman.
6. **Migrasi database**: `SETUP.bat` sudah menjalankannya; atau klik `DB-MIGRATE.bat`. Sebelum perubahan pertama, V2 membuat backup `pre-migration` di `backups\`.
   Migrasi yang diterapkan: `002` (admin tunggal), `003` (status order), `004` (katalog), `005` (tabel riwayat), `006` (trigger immutability). `001` adalah baseline V1 dan dianggap sudah ada.
7. **Periksa**: klik dua kali `CHECK-DATABASE.bat`. Harus berakhir "database SEHAT"; semua file Lua harus "cocok dengan SHA-256".
8. **Jalankan**: `START-BACKEND.bat`, `START-TUNNEL.bat`, lalu login di `/admin` dengan username dan password admin V1 Anda.
   Admin utama adalah **owner tertua** dari V1; password dan 2FA-nya tetap berlaku (selama `ADMIN_SESSION_SECRET` tidak diganti).
9. Buka **Settings** di dashboard dan pastikan kontak Discord benar. Deploy ulang website (`DEPLOY-NETLIFY.bat`) agar halaman dan Function V2 terpasang.

## Apa yang terjadi pada data lama

| Data V1 | Hasil di V2 |
|---|---|
| Admin utama (owner tertua) | Tetap, menjadi administrator tunggal |
| Admin lain | **Dipindah** ke `admins_legacy` (tidak bisa login), sesi mereka dicabut. Datanya tetap ada |
| Order `approved` | Menjadi `paid` (`paid_at` = waktu keputusan); item, pembayaran, dan lisensi tetap utuh |
| Pembayaran `confirmed` | Tetap |
| Versi produk lama | Menjadi `published` (immutable); payload dan SHA-256 tetap |
| Lisensi, entitlement, pengguna, perangkat | Tidak berubah |
| Riwayat di `audit_logs` | Tetap; sebagian disalin ke tabel riwayat baru (`users.last_login_at` dst.) |
| Produk status `draft`/`disabled`/`maintenance` | Tidak berubah |

Pembuktian: `server/tests/v2-ops.test.js` membuat database **berskema V1** (`001_init.sql` dari ZIP V1) berisi dua admin, sesi, produk, lisensi, order `approved`, dan pembayaran,
lalu menjalankan migrasi V2 dan memeriksa semua baris di atas, termasuk backup `pre-migration`, penolakan admin ketiga oleh database, dan penolakan perubahan versi lama.
**Yang belum teruji:** database V1 produksi Anda yang sebenarnya (isinya tidak saya miliki). Karena itu langkah 2 dan `CHECK-DATABASE.bat` pada langkah 7 penting.

## Bila ada yang salah (rollback)

* Migrasi hanya maju. Jalan mundur adalah **memulihkan salinan**: tutup V2, lalu jalankan lagi V1 dari folder V1 asli (yang tidak Anda timpa) dengan salinan manual pada langkah 2.
* Atau di V2: `RESTORE-DATABASE.bat` memilih backup `pre-migration` (database sebagaimana sebelum migrasi). Backend V2 harus berhenti saat restore.
* Jangan mengedit file migrasi lama untuk "memperbaiki": checksum akan menolaknya (server tidak mau start). Perbaikan = file migrasi baru.

## Setelah upgrade

* Pelanggan lama tidak perlu mendaftar ulang; lisensi dan perangkat tetap berlaku.
* URL loader `/loader/<slug>` tidak berubah. Browser kini melihat halaman produk V2; executor tetap mendapat bootstrap.
* Perintah `.bat` baru: `STOP-SERVER.bat`, `RESTORE-DATABASE.bat`, `CHECK-DATABASE.bat`, `BACKUP-ENV.bat`, `RESTORE-ENV.bat`, `RESET-ADMIN-2FA.bat`.
