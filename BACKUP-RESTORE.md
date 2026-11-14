# ISID Hub v1.0.0: Backup dan Restore

Kehilangan `data\isid.db` berarti kehilangan semua pengguna, lisensi, dan order. Backup adalah satu-satunya jalan kembali. Baca dokumen ini sekali, lalu **coba restore sekali** (bagian 6) sebelum Anda benar-benar membutuhkannya.

## 1. Apa yang dicadangkan, dan apa yang tidak

| Item | Dalam backup biasa? | Keterangan |
|---|---|---|
| Database SQLite | **Ya** | Snapshot konsisten via `VACUUM INTO` (aman saat backend hidup), diverifikasi `integrity_check` |
| Script Lua privat (`storage/private/`) | Ya (kecuali `--no-storage`) | Disalin ke `<backup>.storage/` |
| Bukti pembayaran (`storage/proofs/`) | Ya (kecuali `--no-storage`) | Sama |
| Manifest (`.json`) | Ya | Ukuran, SHA-256, versi migrasi, jumlah file, waktu, jenis |
| **`.env` (secret)** | **TIDAK, sengaja** | Backup biasa mungkin disalin ke flashdisk/cloud; secret tidak boleh ikut. Gunakan `BACKUP-ENV.bat` (terenkripsi) |
| `logs/` | Tidak | Log bukan data bisnis |
| Berkas Netlify | Tidak | Ada di akun Netlify Anda; deploy ulang dengan `DEPLOY-NETLIFY.bat` |

Format di folder `backups\`: `isid-YYYYMMDDTHHMMSSZ-<jenis>.db` + `.json` + `.storage\`. Waktu dalam UTC.

## 2. Jenis backup dan retensi

| Jenis | Dibuat oleh | Dipangkas otomatis? |
|---|---|---|
| `manual` | Anda: `BACKUP-DATABASE.bat` atau dashboard -> Database -> Create backup | **Tidak pernah** |
| `auto` | Backend, berkala (bawaan tiap 24 jam; dicek tiap 10 menit) | Ya: yang terbaru dipertahankan sebanyak `backup_retention_count` (bawaan 14) |
| `pre-migration` | Otomatis sebelum migrasi pada database yang sudah berisi data | Ya: 5 terbaru dipertahankan |
| `pre-restore` | Otomatis tepat sebelum restore | Ya: 5 terbaru dipertahankan |

Pengaturan (dashboard -> Settings): `backup_interval_hours`, `backup_retention_count`. Matikan backup otomatis dengan `ISID_AUTO_BACKUP=false` di `.env`.
Catatan: backup otomatis hanya berjalan **saat backend hidup**. Bila PC sering mati, jalankan backup manual secara rutin.

**Salinan luar:** isi `ISID_BACKUP_EXTERNAL_PATH` (mis. `E:\ISID-Backup` atau folder OneDrive/Google Drive yang disinkronkan) agar setiap backup juga disalin ke sana. Hasilnya (OK/GAGAL) ditampilkan di keluaran CLI dan Server Monitor. Backup yang hanya ada di satu disk **bukan** backup.

## 3. Membuat backup

* **Klik dua kali** `BACKUP-DATABASE.bat` (atau VS Code: Terminal -> Run Task -> `ISID: Backup Database`). Boleh dilakukan saat backend hidup.
* **Dari dashboard:** Admin -> Database -> Create backup. Ada tombol *Verify* untuk memeriksa ulang backup lama.
* Opsi CLI: `node server/scripts/backup.js --no-storage` (hanya database) atau `--kind auto`.
* Backup hanya dibuat bila file database **sudah ada** (tidak membuat database kosong baru).

Hasil CLI yang benar menampilkan `Verifikasi : LULUS (integrity_check ok)`. Bila `GAGAL`, jangan mengandalkan backup itu; ulangi.

## 4. Backup `.env` (terenkripsi)

1. Klik dua kali `BACKUP-ENV.bat`, ketik **passphrase** (minimal 8 karakter; tampil di layar karena ini komputer Anda). Ketik dua kali.
2. Hasil: `backups\env\env-<waktu>.enc.json`, dienkripsi scrypt + AES-256-GCM. Tanpa passphrase tidak bisa dibuka. **Simpan passphrase di pengelola password.** Jika hilang, cadangan tidak bisa dipulihkan.
3. Pulihkan dengan `RESTORE-ENV.bat` (pilih file, ketik passphrase). `.env` yang ada **tidak dihapus**: dipindah menjadi `.env.before-restore-<waktu>`.
4. Passphrase salah -> `.env` tidak diubah.

Tanpa `.env` yang sama (terutama `ADMIN_SESSION_SECRET`), 2FA admin tidak bisa dibaca dan Netlify tidak cocok dengan `ISID_GATEWAY_SECRET`. Lakukan `BACKUP-ENV.bat` setelah `SETUP.bat`, dan setiap kali `.env` berubah.

## 5. Memeriksa kesehatan

`CHECK-DATABASE.bat` (aman, tidak mengubah apa pun, boleh saat backend hidup) memeriksa: `integrity_check`, `foreign_key_check`, status migrasi, jumlah baris tiap tabel, tepat satu administrator, dan **SHA-256 serta ukuran setiap file Lua** terhadap database. File di storage yang tidak terdaftar dilaporkan sebagai catatan (tidak dihapus otomatis).
Kode keluar 0 = sehat, 1 = ada masalah.

## 6. Restore

Restore **tidak pernah otomatis** dan selalu membuat salinan pengaman lebih dulu.

### Cara A: dari komputer server (disarankan untuk pemulihan darurat)

1. `STOP-SERVER.bat` (restore ditolak selama backend masih berjalan).
2. Klik dua kali `RESTORE-DATABASE.bat`. Daftar backup tampil; pilih nomornya.
3. Backup **diverifikasi** dulu; bila gagal verifikasi, restore ditolak.
4. Ketik `RESTORE` (huruf besar). Selain itu dibatalkan, tidak ada yang diubah.
5. Database saat ini disalin dulu sebagai backup `pre-restore` (hanya database, tanpa storage), lalu diganti dan dicek ulang.
6. Opsi `--with-storage` (via CLI): mengembalikan file Lua/bukti dari snapshot **hanya bila belum ada** di storage (tidak menimpa file yang ada).
7. `START-BACKEND.bat`. Migrasi yang tertinggal diterapkan otomatis (dengan backup pengaman).

Opsi lain: `--dir <folder>` untuk backup di flashdisk (disalin dulu ke folder backup utama agar salinan pengaman tidak tertulis di flashdisk), `--name <file.db>` dan `--confirm RESTORE` untuk non-interaktif.

### Cara B: dari dashboard (backend hidup)

1. Admin -> Database -> pilih backup -> **Restore**. Wajib: password admin (+ kode 2FA bila aktif) dan mengetik `RESTORE`. Dibatasi 5 kali / 10 menit.
2. Restore **dijadwalkan** (`data/pending-restore.json`), lalu backend restart (kode keluar 75; `START-BACKEND.bat` memulai ulang otomatis).
3. Saat start, **sebelum database dibuka**, restore dijalankan dengan backup `pre-restore`. Bila gagal, file jadwal diberi nama `pending-restore.failed-<waktu>` dan backend tetap start dengan database lama (tidak berhenti dan tidak menimpa apa pun).
4. Jadwal bisa dibatalkan sebelum restart (tombol *Cancel pending restore*).

### Setelah restore
* Semua sesi (admin, pelanggan, perangkat) kembali ke keadaan **saat backup**: login sesudah backup perlu diulang; password admin kembali ke yang berlaku saat backup.
* Lisensi/order yang dibuat **setelah** backup hilang dari database aktif. Data itu masih ada di backup `pre-restore` (bisa dibuka lagi dengan restore ke file itu) dan di log audit pada backup tersebut.

## 7. Uji coba restore (lakukan sekali)

1. Buat backup manual. 2. Ubah sesuatu yang mudah dikenali (mis. buat produk uji). 3. `STOP-SERVER.bat`, `RESTORE-DATABASE.bat` ke backup tadi. 4. `START-BACKEND.bat`: produk uji harus hilang. 5. `CHECK-DATABASE.bat`: harus SEHAT. Bila Anda tidak pernah mencobanya, Anda tidak tahu backup Anda berguna.

## 8. Bukti dari test otomatis

* `v2-ops.test.js`: backup nyata (VACUUM INTO, manifest, salinan storage, `.env` tidak ikut), verifikasi mendeteksi kerusakan, restore nyata saat startup lewat **proses sungguhan**, kegagalan restore tidak merusak, retensi (manual tidak pernah dihapus).
* `v2-cli.test.js`: `backup.js`, `restore.js` (batal tanpa konfirmasi, konfirmasi salah ditolak, `RESTORE` memulihkan password lama, backup `pre-restore` terbentuk, ditolak saat backend hidup, nama berbahaya ditolak), `check-database.js` (mendeteksi file Lua diubah dan database rusak), `backup-env.js`/`restore-env.js` (tanpa plaintext, passphrase salah ditolak).
* **Belum teruji:** disk penuh saat backup, flashdisk dicabut di tengah penyalinan, dan sinkronisasi OneDrive/Google Drive saat file dipakai. Jangan menaruh `data\` itu sendiri di folder yang disinkronkan; taruh **salinan backup** di sana.
