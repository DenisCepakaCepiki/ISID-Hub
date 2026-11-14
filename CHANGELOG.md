# Catatan perubahan: ISID Hub

## v1.0.0 (rilis gabungan)

Satu proyek tunggal hasil penggabungan **V2.0 (milik Anda, dikembangkan di sini)** dan **V2.1 "OneClick" (dari AI lain)**, ditambah fitur baru. Nomor versi **dikembalikan ke 1.0.0** sesuai permintaan Anda; angka "V2.0/V2.1" di komentar kode lama hanya nama internal tahap pengembangan.

Prinsip penggabungan: integrasi minimal dan aman. Sistem inti (produk dinamis, versi immutable, isolasi per produk, admin tunggal, backup, gateway Netlify) **tidak ditulis ulang**. Bagian V2.1 yang bagus diambil, bagian yang menurunkan keamanan atau bertabrakan dibuang.

### Dari V2.1 OneClick (dipertahankan)
* `START-ISID.bat`, `STOP-ISID.bat`, `STATUS-ISID.bat`: satu klik, **idempoten** (tidak membuat backend/tunnel ganda), ringkasan jelas.
* Cloudflare Tunnel sebagai **Windows Service `Cloudflared`** (remotely-managed, hostname permanen dari *Published application route*), bukan jendela CMD. `SETUP-TUNNEL.bat` mengisi `ISID_BACKEND_ORIGIN`. Token tunnel **tidak pernah** dibaca/disimpan/diminta.
* Alat operasi `server/scripts/ops.js`, penjaga port/duplikat di `START-BACKEND.bat`, keluar normal pada skrip pemeriksaan (menghindari crash libuv Node di Windows).
* Kebijakan **executor yang diizinkan** (`enforce` / `monitor` / `off`; bawaan Delta, Real, Madium) + migrasi `007`.

### Bug yang diperbaiki (dari log nyata Anda)
* `DEPLOY-NETLIFY.bat`: status login dibaca dari isi keluaran Netlify CLI (bukan kode keluar); `env:set` untuk variabel yang sudah ada tidak lagi ditolak (konteks + scope dipasang lewat perintah terpisah); peringatan DEP0190 hilang; hasil dibaca dari file hasil, bukan kode keluar Node.
* `START-BACKEND.bat`: backup otomatis gagal dengan `EPERM: fsync` di Windows. Penyebab: `fsync` pada handle read-only. Perbaikan: buka `r+` dan jadikan best-effort (tes `hub-fixes`).

### Baru: akun, login, persetujuan
* **Login = username Roblox + password ISID.** Verifikasi frasa di profil Roblox, pairing code, dan mode "weak" **dihapus** (kolom/tabel lama dibiarkan di database; tidak dipakai).
* Akun **dibuat admin** (tidak ada pendaftaran mandiri). Password sementara acak tampil **sekali**; pelanggan **wajib menggantinya** saat login pertama. Reset password, buka kunci, hapus akun dari dashboard admin. Password di-hash (scrypt/Argon2id), kunci 15 menit setelah 8 kali salah, pembatas per username dan IP, pesan galat sama untuk "username tidak ada" dan "password salah".
* **Loader** memakai kredensial yang sama. Urutan pemeriksaan di server: (1) username pemain yang sedang menjalankan = username terdaftar, (2) password, (3) UserId terdaftar, (4) password sementara sudah diganti, (5) lisensi **per produk**. Setiap hasil dicatat dan dikirim ke webhook.
* **Terms of Service** (8 bagian + Developer Statement), **Privacy Policy**, **Contact Support** di `/legal`; pop-up **"Saya setuju"** sebelum login dan sesudah login (versi dan waktu setujunya disimpan di server). Teks adalah **draf**: periksa dan sesuaikan (`legal-content.js`).
* Pengunjung belum login melihat tutorial `loadstring`; tanpa tombol daftar; tertulis "Belum punya akun? Hubungi atau Masuk Discord Isekai Station ID".

### Baru: key trial gratis
* Format `ISID-XXXX-XXXX-XXXX`, **dibuat hanya oleh admin** dan terlihat di Admin -> Trial Keys. Tanpa login di loader. Pengaturan: produk, lama trial, batas jumlah klaim, masa berlaku key, satu akun Roblox hanya sekali per script, batas per IP, cabut/pulihkan. Seluruh klaim tercatat.

### Baru: Discord webhook + log eksekusi
* Webhook **OWNER** (privat, status + pelacakan, menyebut ID Discord `631091920654958648`) dan **PUBLIC** (pengumuman, "user telah membeli" hanya username Roblox, bisa disembunyikan per akun). 11 slot bawaan (8 OWNER + 3 PUBLIC) + webhook kustom per channel. Tiap webhook: URL (diisi dari web admin, **disimpan terenkripsi**, tidak pernah ditampilkan lagi), saklar aktif, daftar event, tombol **Send test**. Pengiriman ulang untuk 429/5xx; kegagalan Discord tidak pernah menggagalkan aksi utama.
* Bawaan **tanpa password** di webhook. Password sementara hanya ikut bila Anda menyalakan opsinya sendiri (hanya OWNER).
* **Admin -> Executions**: setiap percobaan loader (berhasil maupun ditolak) dengan username Roblox, UserId, executor, device id, HWID (hash), IP, map/game; ringkasan, filter, refresh langsung. Disimpan 90 hari (bisa diatur) lalu dihapus otomatis.

### Baru: tampilan dan bahasa
* Tema **gradasi hitam (atas) ke magenta `#FF00FF` (bawah)** dengan kaca cair ala iOS; wallpaper dan logo animasi milik Anda **opsional** (`docs/PANDUAN-TAMPILAN.md`).
* **Dua bahasa**: Indonesia (utama) dan English; tombol ID | EN. Dashboard admin tetap Inggris.

### Baru: katalog
* Kartu **"Update the script (new version)"** (publish + jadikan aktif sudah tercentang) dan tombol **"Roll back to vX"** satu klik.

### Migrasi database
* `008_hub_v1.sql` (baru; **hanya menambah**): kolom akun di `users`, tabel `execution_logs`, `trial_keys`, `trial_claims`, `discord_webhooks`. Migrasi 001-007 tidak berubah (dijaga checksum). Backup `pre-migration` otomatis dibuat sebelum diterapkan pada database yang sudah berisi.
* **Pelanggan lama tidak punya password** setelah upgrade: admin perlu **Reset password** per pelanggan (lihat `PANDUAN-SETELAH-TERPASANG.md`).

### Dibuang
* Frasa profil/pairing code/mode weak; `allow_registration` dan `verification_mode` di Settings; peringatan "securityLevel" di Overview.
* `INSTALL-ID.txt` lama (digantikan `PANDUAN-INSTALASI-AWAL.md` dan `PANDUAN-SETELAH-TERPASANG.md`); `docs/LAPORAN-AKHIR.md` (digantikan `docs/LAPORAN-PENGUJIAN.md`).

### Pengujian
**332 test otomatis, 332 lulus, 0 gagal, 0 dilewati**; uji tampilan Chromium untuk halaman pelanggan dan admin lulus. **Belum teruji:** `.bat` di Windows, Cloudflare, Netlify CLI asli, Roblox/executor/Discord asli. Rincian: `docs/LAPORAN-PENGUJIAN.md`.

---

## Riwayat tahap sebelumnya (nomor internal lama, sudah tercakup di v1.0.0)

* **2.1.0** (OneClick, AI lain): kebijakan executor, migrasi `007`, `START/STOP/STATUS-ISID`, service Cloudflared.
* **2.0.1**: perbaikan deteksi login `DEPLOY-NETLIFY.bat` dari log nyata.
* **2.0.0**: multi-produk dinamis, admin tunggal, pembayaran manual via Discord, backup/restore, dashboard admin.
