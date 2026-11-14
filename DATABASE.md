# ISID Hub v1.0.0: Database

Database: **SQLite** (modul bawaan `node:sqlite`, tanpa paket npm). File: `data/isid.db` (atau sesuai `DATABASE_URL=sqlite:<path>`).
Mode WAL, `foreign_keys=ON`, transaksi tulis memakai `BEGIN IMMEDIATE`. PostgreSQL **tidak didukung**.

Semua `id` berupa UUID teks. Semua waktu berupa **epoch milidetik UTC** (`BIGINT`). Harga berupa **bilangan bulat** satuan terkecil mata uang (rupiah).
Daftar di bawah dihasilkan dari menjalankan kedelapan migrasi pada database kosong (bukan ditulis dari ingatan).

## 1. Tabel (26 termasuk `schema_migrations`)

| Tabel | Fungsi | Kolom penting |
|---|---|---|
| `admins` | Administrator **tunggal** | `username`, `password_hash` (`$scrypt$...` atau argon2id), `totp_secret_enc` (terenkripsi), `totp_enabled`, `totp_last_step` (anti-replay), `failed_logins`, `locked_until` |
| `admins_legacy` | Arsip admin tambahan dari V1 (tidak bisa login) | `archived_at`, `archive_reason` |
| `users` | Pelanggan (akun Roblox) | `roblox_user_id`, `roblox_username`, `status` (`active`/`suspended`/`banned`), `last_login_at`, `password_hash`, `must_change_password`, `failed_logins`, `locked_until`, `hide_public_name`, `terms_version`, `terms_accepted_at` |
| `execution_logs` | Jejak setiap percobaan loader (berhasil/ditolak) | `event`, `result`, `reason`, `login_username`, `roblox_user_id/username` (dilaporkan klien), `executor`, `device_id`, `hwid_hash`, `place_id/game_id/job_id`, `ip`; dihapus otomatis setelah N hari |
| `trial_keys` | Key trial gratis `ISID-XXXX-XXXX-XXXX` | `key_code` unik, `product_id`, `claim_minutes`, `max_claims`, `valid_until`, `status` |
| `trial_claims` | Klaim key trial | `UNIQUE(trial_key_id, roblox_user_id)`, `expires_at`, `uses`, `ip`, `executor` |
| `discord_webhooks` | Tujuan webhook Discord (OWNER/PUBLIC) | `scope`, `url_enc` (terenkripsi), `enabled`, `events`, `mention_owner`, status kirim terakhir |
| `products` | Produk dinamis | `slug` unik, `status` (`draft`/`active`/`disabled`/`maintenance`), `visibility` (`public`/`unlisted`/`private`), `price_amount`, `license_duration_days`, `current_version_id`, `archived_at`, `tags`, `features` (JSON teks) |
| `product_versions` | Versi produk | `version`, `release_status` (`draft`/`published`), `changelog`, `published_at` |
| `product_payloads` | Metadata file Lua per versi | `storage_path`, `sha256`, `size_bytes`, `original_name`, `content_type` |
| `licenses` | Lisensi per (user, produk) | `status` (`pending`/`active`/`expired`/`suspended`/`revoked`), `starts_at`, `expires_at` (NULL = lifetime), `order_id`, `device_limit` |
| `entitlements` | Hak akses aktif per (user, produk) | `UNIQUE(user_id, product_id)`, FK komposit ke `licenses` |
| `license_history` | Riwayat setiap perubahan lisensi | `event`, `status_before/after`, `expires_before/after`, `actor_*` |
| `orders` | Pesanan | `reference` unik, `status` (`pending`/`paid`/`rejected`/`cancelled`/`refunded`), `subtotal_amount`, `discount_amount`, `total_amount`, `paid_at`, `refunded_at` |
| `order_items` | Isi pesanan | `product_id`, `product_slug`, `product_name`, `unit_price`, `duration_days`, `license_id` |
| `payments` | Catatan pembayaran | `provider` (`manual`), `status` (`pending`/`confirmed`/`failed`/`refunded`), `method_label` |
| `payment_history` | Riwayat pembayaran | `event`, `status_before/after`, `amount`, `actor_*` |
| `payment_attachments` | Bukti pembayaran | `storage_path`, `sha256`, `size_bytes`, `original_name` (disanitasi) |
| `payload_deliveries` | Log setiap pengiriman Lua | `result`, `reason`, `trace` (watermark), `version`, `ip_hash` (bukan IP asli), `executor` (nama yang dilaporkan klien; NULL = tidak melapor atau baris lama) |
| `notifications` | Notifikasi pelanggan | `kind`, `title`, `body`, `read_at` |
| `webhook_events` | Disiapkan untuk gateway pembayaran di masa depan (belum dipakai) | `provider`, `event_id`, `signature_ok`, `status` |
| `sessions` | Sesi dan token | `kind` (`admin`/`web`/`device`/`access`), `token_hash` (SHA-256; token asli tidak disimpan), `expires_at`, `revoked_at` |
| `auth_challenges` | (LEGACY, tidak dipakai lagi sejak v1.0.0) challenge frasa profil | dibiarkan agar data lama utuh |
| `audit_logs` | Jejak audit | `actor_*`, `action`, `entity_*`, `ip`, `meta` |
| `system_settings` | Pengaturan (key/value) | lihat bagian 5 |
| `schema_migrations` | Catatan migrasi | `name`, `applied_at`, `checksum` |

## 2. Aturan yang dijaga langsung oleh database

| Aturan | Mekanisme |
|---|---|
| **Hanya satu administrator** | Indeks unik berekspresi `idx_admins_single` pada `((1))` |
| **Akses produk A tidak membuka produk B** | `entitlements` FK komposit `(license_id, user_id, product_id) -> licenses(id, user_id, product_id)` + `UNIQUE(user_id, product_id)` |
| **Versi published tidak bisa berubah** | Trigger `trg_version_published_immutable`, `trg_payload_published_immutable` |
| **Versi/payload published tidak bisa dihapus** (selama produknya ada) | Trigger `trg_version_published_nodelete`, `trg_payload_published_nodelete` |
| **Publish wajib punya payload** | Trigger `trg_version_publish_needs_payload` |
| **`current_version_id` harus versi published milik produk itu** | Trigger `trg_product_current_must_be_published` |
| **Status valid** | Batasan `CHECK` pada status user, produk, versi, lisensi, order, pembayaran, sesi |
| **Diskon tidak melebihi subtotal** | `CHECK (discount_amount <= subtotal_amount)` |
| **Slug/referensi unik** | `UNIQUE` pada `products.slug`, `orders.reference`, dll. |

Pesan kesalahan trigger: `published_version_immutable` dll. Server mengubahnya menjadi respons yang jelas, tetapi bila seseorang mengedit database
langsung (mis. dengan alat SQLite), database tetap menolak. Ini diuji (`v2-catalog.test.js`).

## 3. Transaksi

Operasi multi-langkah berjalan dalam satu transaksi (`BEGIN IMMEDIATE`): approve order (ubah status + buat lisensi + entitlement + riwayat + notifikasi),
publish versi, rollback, refund, restore pengaturan. Approve **idempoten**: dua permintaan paralel menghasilkan satu lisensi (diuji).

## 4. Migrasi

File ada di `server/src/db/migrations/`, dijalankan berurutan oleh `core` saat startup atau `DB-MIGRATE.bat`.

| File | Isi |
|---|---|
| `001_init.sql` | Skema dasar V1 (tidak boleh diedit) |
| `002_single_admin.sql` | Satu admin. Admin tambahan **dipindah** (disalin lalu dihapus dari `admins`) ke `admins_legacy`; sesi mereka dicabut; indeks unik satu-admin |
| `003_orders_v2.sql` | Status order V2 (`approved` V1 menjadi `paid`), `subtotal/discount`, `paid_at`, `refunded_at` (tabel dibangun ulang; data tetap) |
| `004_catalog_v2.sql` | Kolom katalog (visibility, tags, features, arsip), `release_status`, `device_limit`, kolom order item |
| `005_history_tables.sql` | `license_history`, `payload_deliveries`, `payment_attachments`, `payment_history`, `notifications`, `webhook_events`; riwayat lama disalin dari `audit_logs` |
| `006_immutable_versions.sql` | Trigger immutability dan integritas versi |
| `007_executor_gate.sql` | Kolom `payload_deliveries.executor` + indeks (kebijakan executor). Hanya menambah kolom; data lama tidak berubah |
| `008_hub_v1.sql` | Login password (kolom di `users`), `execution_logs`, `trial_keys`, `trial_claims`, `discord_webhooks`, persetujuan ToS. Hanya **menambah**; pelanggan lama tidak punya password sampai admin me-reset |

**Aturan migrasi:**

1. **File yang sudah pernah diterapkan tidak boleh diedit.** `schema_migrations.checksum` (SHA-256 dengan akhir baris LF dinormalisasi) membuat server menolak start bila ada file lama yang berubah.
2. Perubahan skema = file baru bernomor berikutnya (`009_...sql`).
3. Bila database sudah berisi migrasi, **backup `pre-migration` dibuat otomatis** sebelum migrasi baru diterapkan.
4. Direktif `-- isid:foreign-keys-off` pada baris pertama (dipakai 002 dan 003) mematikan FK selama rebuild tabel, lalu `foreign_key_check` dijalankan; migrasi gagal bila ada pelanggaran.
5. Migrasi dari database V1 nyata diuji (`v2-ops.test.js`): data user, produk, lisensi, order, dan pembayaran tetap utuh.

## 5. `system_settings` (dapat diubah dari Admin -> Settings)

| Key | Bawaan | Keterangan |
|---|---|---|
| `site_name` | Isekai Station ID | |
| `maintenance_mode`, `maintenance_message` | `0` | Mode perawatan global (diatur dari Server Monitor) |
| `allow_registration`, `verification_mode` | - | (LEGACY, tidak dipakai lagi; tidak ada pendaftaran mandiri dan tidak ada verifikasi frasa) |
| `trial_enabled` | `1` | Key trial gratis aktif/nonaktif |
| `execution_log_retention_days` | `90` | Log eksekusi dihapus otomatis setelah N hari (1-3650) |
| `discord_owner_id` | `631091920654958648` | ID Discord pemilik untuk @mention di webhook OWNER |
| `discord_include_temp_password`, `public_show_username` | `0` / `1` | Opsi webhook: password sementara ikut tampil di OWNER (bawaan mati); nama Roblox tampil di pengumuman PUBLIC |
| `max_devices`, `device_session_days`, `access_token_ttl_sec`, `web_session_hours` | 3, 30, 300, 168 | Batas perangkat dan umur sesi |
| `community_name`, `discord_username`, `discord_invite_url` | Isekai Station ID, `denis.cepaka`, `https://discord.gg/isekai-station-id-703563738367655937` | Kontak Discord (tampil di semua halaman relevan) |
| `payment_instructions`, `support_contact` | teks bawaan | Instruksi pembayaran manual (tanpa nomor bank/e-wallet) |
| `max_lua_bytes`, `max_proof_bytes` | 2 MB | Maksimum 4 MB (batas body Netlify Function) |
| `refund_revokes_license` | `1` | Refund order mencabut lisensi dari order itu |
| `backup_retention_count`, `backup_interval_hours` | 14, 24 | Retensi dan interval backup otomatis |

## 6. Penyimpanan file (di luar database)

| Lokasi | Isi |
|---|---|
| `storage/private/<slug>/v<versi>.lua` | Lua privat. Tidak pernah disajikan oleh web statis. |
| `storage/proofs/...` | Bukti pembayaran (nama disanitasi, tidak ada path dari pengguna) |
| `storage/trash/` | File yang diganti/dihapus dipindah ke sini, **tidak dihapus permanen** |
| `backups/` | Backup database (`*.db` + `*.json` manifest + `*.storage/`) dan `backups/env/` (`.env` terenkripsi) |

Setiap file Lua tercatat dengan SHA-256 dan ukuran. `CHECK-DATABASE.bat` membandingkan file di disk dengan database dan melaporkan yang berbeda atau hilang.

## 7. Kebijakan data

* **Tidak ada penghapusan data tanpa jejak.** Produk yang pernah punya lisensi/order tidak bisa dihapus (hanya diarsipkan). `license_history` sengaja tanpa FK ke `licenses` sehingga riwayat tetap ada walau baris lisensinya hilang; `payment_history` terikat ke order (order tidak dihapus).
* Token (sesi, device key, access token) hanya disimpan sebagai hash. IP pada `payload_deliveries` disimpan sebagai hash.
* Rahasia (`.env`) tidak ada di database. Secret 2FA disimpan terenkripsi dengan kunci turunan `ADMIN_SESSION_SECRET`.
