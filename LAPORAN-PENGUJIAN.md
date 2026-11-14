# Laporan pengujian: ISID Hub v1.0.0

Dikembangkan oleh Denis Cepaka, Claude & ChatGPT. Laporan ini hanya memuat hasil yang **benar-benar dijalankan**; bagian "BELUM teruji" sama pentingnya dengan bagian "lulus".

## 1. Test otomatis (dijalankan pada pohon proyek final)

`npm test` = `node --test --test-concurrency=1 server/tests/*.test.js` di Linux (Node 22, SQLite bawaan Node, interpreter Lua 5.4 tersedia).

| Hasil | Nilai |
|---|---|
| Total test (termasuk subtest) | **332** |
| Lulus | **332** |
| Gagal | 0 |
| Dilewati | 0 |
| Durasi | ~275 detik |

Isi suite (satu berkas = satu topik):

| Berkas | Yang dibuktikan |
|---|---|
| `hub-auth` | Akun dibuat admin; login username + password; password sementara tampil sekali dan **wajib diganti**; lockout 8 gagal = 15 menit; reset password mencabut sesi dan device key; password lama tidak berlaku; persetujuan ToS/Privacy per versi; verifikasi username pemain **sebelum** lisensi; log eksekusi (berhasil/ditolak); tanpa jalur frasa profil |
| `hub-trial` | Key `ISID-XXXX-XXXX-XXXX`: dibuat hanya admin, klaim pertama menetapkan masa trial, batas klaim/masa berlaku/IP, cabut/pulihkan, tidak bisa dihapus bila sudah diklaim, trial bisa dimatikan |
| `hub-discord` | 11 slot webhook (8 OWNER, 3 PUBLIC); URL **terenkripsi** dan tidak pernah dikembalikan API; toggle per webhook; tombol test; mention ID Discord; event ke tujuan yang benar; owner menerima detail, publik hanya username Roblox; kegagalan Discord tidak mengganggu permintaan utama; password sementara tidak dikirim ke webhook kecuali dinyalakan |
| `hub-frontend` | Sintaks semua JS frontend; versi hukum frontend = server; ToS 8 bagian + Developer Statement; kamus Indonesia dan terjemahan galat; tidak ada sisa alur lama (frasa profil/pairing); halaman pelanggan dua bahasa, admin tetap Inggris; `/legal` dilayani |
| `hub-fixes` | Backup tahan terhadap `fsync` yang ditolak Windows (EPERM, bug nyata dari log Anda) |
| `integration`, `lua-e2e` | Skenario isolasi produk A-G; bootstrap Lua dijalankan di **interpreter Lua sungguhan** dengan mock Roblox/executor terhadap server sungguhan |
| `security` | Gateway secret, CSRF, rate limit, TOTP (vektor RFC 6238), path traversal, token ter-hash, frontend tanpa secret |
| `v2-catalog`, `v2-orders`, `v2-system`, `v2-ops`, `v2-cli`, `v2-executor` | Versi draft/published + immutability (trigger), rollback, order idempoten, monitoring jujur, backup/restore nyata, migrasi dari database lama, skrip CLI nyata (termasuk terminal pty), kebijakan executor |
| `deploy`, `netlify` | Skrip deploy dengan Netlify CLI **tiruan** (tidak pernah menggantung); gateway Function terhadap backend sungguhan |
| `v21-bat`, `v21-ops` | Pemeriksaan **statis** file `.bat` (bukan menjalankannya) dan alat operasional (`ops.js`, `setup-tunnel.js`) dengan backend sungguhan |

## 2. Uji tampilan di Chromium sungguhan

Dijalankan saat pengembangan (Playwright + Chromium) terhadap backend uji sungguhan (Discord dan Roblox ditiru). Skrip ujinya sementara dan **tidak ikut paket**. Hasil: lulus.

* **Halaman pelanggan:** landing (tutorial `loadstring`, info key trial, "Belum punya akun? Hubungi atau Masuk Discord", tanpa tombol daftar), `/legal` (8 bagian ToS), pop-up persetujuan sebelum dan sesudah login (tidak bisa ditutup tanpa memilih), login dengan password salah (pesan Indonesia), wajib ganti password sementara, tombol ID/EN (pilihan bertahan setelah muat ulang), tampilan ponsel 390 px tanpa gulir horizontal.
* **Dashboard admin:** buat akun (password sementara sekali tampil), reset password (password lama langsung tidak berlaku), tabel Executions (termasuk percobaan ditolak), Trial Keys (buat, cabut, pulihkan, hapus, klaim), Discord Webhooks (8 + 3 kartu, simpan URL, kirim test ke Discord tiruan, saklar, URL tidak pernah kembali), Settings, kartu "Update the script" dan tombol rollback.
* Satu-satunya galat konsol: `404` untuk `logo.gif`/`wallpaper.png` yang **memang belum Anda taruh** (opsional).
* **Catatan:** gambar wallpaper dan GIF Anda tidak diuji secara visual di tema (tidak ikut paket); tema diuji dengan gradasi saja. Periksa tampilannya setelah Anda menaruhnya (`docs/PANDUAN-TAMPILAN.md`).

## 3. Yang BELUM teruji (jangan anggap berfungsi sebelum Anda mencobanya)

| Hal | Alasan | Cara membuktikan |
|---|---|---|
| File `.bat` (`START-ISID`, `STOP-ISID`, `STATUS-ISID`, `SETUP`, `DEPLOY-NETLIFY`, dst.) di **Windows** | Lingkungan pembuat Linux; hanya logika Node di belakangnya dan pemeriksaan statis `.bat` yang diuji. Path berspasi, UAC, Task Scheduler belum terbukti | `docs/UJI-MANUAL.md` bagian 1 |
| **Cloudflare**: service `Cloudflared`, Published application route, hostname permanen | Tidak ada akses akun Cloudflare Anda. (Tunnel Anda masih 0 route saat tangkapan layar: route harus Anda buat) | Bagian 3 |
| **Netlify CLI asli** dan CDN Netlify | Hanya CLI tiruan dan Function lokal; langkah [1/4] pernah dicocokkan dengan log nyata Anda | Bagian 4 |
| **Roblox sungguhan**: pencarian username/UserId (Users API) saat admin membuat akun; `Players.LocalPlayer` | Roblox ditiru di test. Bila PC Anda tidak bisa menjangkau Roblox, UserId bisa diketik manual | Bagian 6 |
| **Executor sungguhan** (Delta, dll.) dan Luau asli | Bootstrap hanya dijalankan di Lua 5.4 dengan mock. Fungsi seperti `identifyexecutor`, `gethwid`, penyimpanan file lokal, GUI Roblox tidak terbukti | Bagian 7 |
| **Discord asli** | Webhook diuji terhadap server Discord tiruan | Tombol **Send test** di Discord Webhooks |
| Beban tinggi / kapasitas | Tidak diukur | Mulai kecil, pantau Server Monitor |
| Teks hukum | ToS asli Anda tidak dapat saya pulihkan kata demi kata (hanya ringkasannya); yang ada draf. Bukan nasihat hukum | Periksa/ganti `legal-content.js` (lihat panduan instalasi langkah 11) |
| Audit keamanan pihak ketiga | Tidak ada | Pertimbangkan sebelum skala besar |

## 4. Batas keamanan yang harus Anda ketahui

* **Data dari klien bisa dipalsukan:** nama executor, username Roblox yang dilaporkan, `device id`, dan data `HWID` dikirim oleh script di sisi pemain. Server memverifikasi username pemain **terhadap akun ISID yang login** dan mencatatnya, tetapi pemain yang menguasai klien bisa berbohong. Pakai log sebagai **petunjuk**, bukan bukti hukum. Key trial dibatasi per akun Roblox yang dilaporkan, per IP, masa berlaku, dan jumlah klaim.
* **Script yang sudah dikirim ke pemain berlisensi bisa disalin pemain itu**; sistem membatasi siapa yang menerima, bukan apa yang dilakukan penerima.
* **Webhook:** URL Discord disimpan terenkripsi (AES-GCM, kunci turunan `ADMIN_SESSION_SECRET`). Siapa pun yang memegang URL webhook (mis. bila ditempel ke chat) bisa mengirim pesan ke channel itu: buat ulang webhook di Discord bila bocor. Password sementara **tidak** dikirim ke Discord kecuali opsi *include temporary password* Anda nyalakan sendiri (bawaan: mati, hanya status).
* **Gateway Secret yang pernah tertulis di chat harus dianggap bocor:** ganti (`PANDUAN-SETELAH-TERPASANG.md` bagian 6).
* **Deploy manual satu ZIP penuh ke Netlify** (bila pernah dilakukan) bisa membuat folder `seed/*.lua` dan kode server ikut terbuka di situs lama. Periksa `https://situs-anda/seed/isid-camera-studio.lua` (harus 404), hapus deploy lama di Netlify, dan anggap Lua seed itu sudah terpapar. `DEPLOY-NETLIFY.bat` hanya mengirim `site\public` dan `site\netlify\functions`.

## 5. Jumlah dan ukuran (dihitung dari paket)

| Ukuran | Nilai |
|---|---|
| Endpoint API | 124 (`docs/API.md`, dicek otomatis terhadap router) |
| Migrasi database | 8 (`001`-`008`, hanya maju, dijaga checksum) |
| Dependensi npm backend | 0 |
| Test otomatis | 332 |
