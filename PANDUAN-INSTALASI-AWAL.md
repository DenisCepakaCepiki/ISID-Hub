# Panduan instalasi awal: ISID Hub v1.0.0

Untuk **pertama kali** memasang ISID Hub di PC Windows Anda. Dikerjakan **SEKALI**. Sesudahnya cukup baca `PANDUAN-SETELAH-TERPASANG.md` (pemakaian harian: klik `START-ISID.bat`).

> Sudah pernah memasang V2.0 / V2.1 / ISID Hub sebelumnya dan punya data pelanggan? **Jangan** ikuti panduan ini; buka bagian "Upgrade dari pemasangan lama" di `PANDUAN-SETELAH-TERPASANG.md`.

Tidak perlu bisa pemrograman. Ikuti urutannya. Bila satu langkah gagal, **berhenti**, salin pesan merah/`[GAGAL]` persisnya (tanpa password/secret), dan lihat bagian 12 di bawah.

## Gambaran besarnya

```
Pemain (Roblox executor / browser)
   -> Netlify (website + penghubung)  ->  Cloudflare Tunnel (service Windows "Cloudflared")  ->  Backend Node.js di PC ini  ->  SQLite
```

PC ini harus **menyala dan online** agar pelanggan bisa memuat script. Website tetap tampil walau PC mati, tetapi login dan script tidak.

## Yang harus disiapkan

| Kebutuhan | Keterangan |
|---|---|
| Windows 10/11 | PC yang bisa menyala lama. Matikan Sleep/Hibernate saat dicolok listrik |
| Node.js **22.13 atau lebih baru** (LTS) | Unduh di nodejs.org, pasang dengan semua pilihan bawaan |
| Akun Netlify (gratis) | netlify.com |
| Akun Cloudflare (gratis) **dan sebuah domain** yang DNS-nya dikelola Cloudflare | Untuk alamat tunnel **permanen** (mis. `api.domainmu.com`). Tanpa domain tidak ada alamat tetap |
| `cloudflared` | Dipasang di langkah 5 |

Tidak ada `npm install` untuk backend. Netlify CLI dipasang otomatis oleh `DEPLOY-NETLIFY.bat`.

## Langkah 1: Ekstrak ZIP ke folder sederhana

Klik kanan `ISID-Hub-v1.0.0.zip` -> **Extract All** (ke mana saja, mis. Downloads). Di hasilnya ada satu folder bernama **`ISID-Hub`**.
**Pindahkan folder `ISID-Hub` itu ke `C:\`** sehingga file `C:\ISID-Hub\START-ISID.bat` ada (disarankan: nama folder **tanpa spasi** dan **bukan** di dalam OneDrive/Desktop yang disinkronkan).
Pastikan di dalam `C:\ISID-Hub` langsung ada `START-ISID.bat`, `SETUP.bat`, folder `server`, `site`, `seed` (bukan folder `ISID-Hub` lagi di dalamnya).

## Langkah 2: SETUP.bat (membuat pengaturan rahasia + database)

Klik dua kali **`SETUP.bat`**. Ia memeriksa Node.js, membuat file `.env` (dua secret acak dibuat otomatis), dan menyiapkan database.
Aman diulang: nilai yang sudah ada tidak diubah. **File `.env` adalah kunci sistem: jangan dikirim ke siapa pun** (termasuk ke chat/AI), jangan diunggah ke GitHub.

## Langkah 3: CREATE-ADMIN.bat (akun admin, hanya SATU)

Klik dua kali **`CREATE-ADMIN.bat`**, ketik username dan password admin. Password **tampil saat diketik** di jendela hitam ini (khusus layar lokal Anda; di website tetap tersembunyi).
Minimal 12 karakter. Simpan di tempat aman. Hanya boleh ada satu admin; menjalankannya lagi akan ditolak (itu normal).

## Langkah 4: Siapkan Cloudflare Tunnel (permanen)

Ini bagian paling banyak klik. Tampilan Cloudflare bisa sedikit berbeda dari tulisan di bawah; ikuti nama menunya.

1. Masuk **dash.cloudflare.com** -> **Zero Trust / Cloudflare One** -> **Networks** -> **Tunnels**. Buat tunnel baru (tipe **Cloudflared**) dengan nama bebas, mis. `isid`. Bila sudah ada tunnel `isid`, pakai itu.
2. Pilih sistem operasi **Windows**. Cloudflare menampilkan sebuah perintah pemasangan yang mengandung **token** panjang.
   > **Token itu SAMA dengan password tunnel Anda.** Jangan kirim ke siapa pun, jangan tempel di chat (termasuk chat dengan AI), jangan simpan di folder ISID Hub. ISID Hub **tidak pernah** meminta token itu.
3. Pasang `cloudflared`: buka **Command Prompt sebagai Administrator** (klik Start, ketik `cmd`, klik kanan -> *Run as administrator*) lalu ketik:
   `winget install --id Cloudflare.cloudflared`
4. Di Command Prompt Administrator yang sama, tempel **perintah yang ditampilkan Cloudflare** (bentuknya `cloudflared.exe service install <token>`) lalu Enter. Ini memasang **Windows Service bernama `Cloudflared`** yang menyala otomatis setiap Windows hidup (tidak perlu jendela CMD tunnel).
5. Kembali ke halaman tunnel di Cloudflare: status harus **HEALTHY**.
6. Buka tab **Published application routes** (di versi lama bernama *Public Hostname*) -> **Add a published application route**:
   * **Subdomain**: mis. `api`, **Domain**: pilih domain Anda -> hasil `api.domainmu.com`
   * **Service type**: **HTTP**
   * **URL**: `127.0.0.1:8787` (bila Cloudflare meminta lengkap: `http://127.0.0.1:8787`)
   * Simpan.

## Langkah 5: SETUP-TUNNEL.bat (mengisi hostname ke pengaturan)

Klik dua kali **`SETUP-TUNNEL.bat`**, ketik hostname tadi (mis. `https://api.domainmu.com`, tanpa garis miring di akhir). Ia hanya mengubah satu baris `ISID_BACKEND_ORIGIN` di `.env`.

## Langkah 6: START-ISID.bat (menyalakan semuanya)

Klik dua kali **`START-ISID.bat`**. Ia: memeriksa konfigurasi, membuka backend di jendela **terpisah yang di-minimize (jangan ditutup)**, memastikan service Cloudflared **RUNNING** (Windows mungkin meminta izin Administrator/UAC: pilih *Yes*), dan mengetes jalur hostname ke backend.
Hasil akhirnya berupa RINGKASAN. Harapan: `Backend lokal: OK`, `Cloudflared Service: OK (RUNNING)`, `Hostname tunnel: OK`.
Aman diklik berulang kali: tidak membuat backend atau tunnel ganda.

## Langkah 7: DEPLOY-NETLIFY.bat (memasang website)

Klik dua kali **`DEPLOY-NETLIFY.bat`**. Pertama kali: Netlify CLI dipasang (jawab `Y`), browser terbuka untuk login Netlify, lalu skrip menghubungkan site, memasang variabel `ISID_BACKEND_ORIGIN` dan `ISID_GATEWAY_SECRET` (nilai secret **tidak dicetak**), lalu deploy. Setiap langkah punya batas waktu dan tidak menggantung diam; hasilnya berupa RINGKASAN `[OK]/[GAGAL]`.
Hanya folder `site\public` dan `site\netlify\functions` yang dikirim ke Netlify: **folder `seed`, `server`, `storage`, `.env` tidak ikut** (jangan pernah men-drag seluruh folder/ZIP ke Netlify secara manual).

## Langkah 8: Periksa

1. Klik **`STATUS-ISID.bat`** (hanya membaca, tidak mengubah apa pun) dan **`TEST-SERVER.bat`**. Semua `[OK]`.
2. Buka website Anda (mis. `https://isekaistationid.netlify.app`). Tampil landing berbahasa Indonesia dengan tutorial `loadstring`.
3. Buka `/admin`, login dengan akun admin langkah 3.
4. Cek bahwa script mentah tidak terbuka: di browser buka `https://alamat-netlify-anda/seed/isid-camera-studio.lua` (atau nama file seed apa pun): harus **404**.

## Langkah 9: Isi dashboard admin (sekali)

Di `/admin` (semua dalam bahasa Inggris):

| Menu | Yang dilakukan |
|---|---|
| **Settings** | Pastikan kontak Discord benar. Di "Free trial and logs": trial key aktif/tidak dan lama log eksekusi disimpan (bawaan 90 hari) |
| **Discord Webhooks** | Isi **Your Discord user ID** (`631091920654958648`), lalu tempel URL webhook di kartu **OWNER** (privat) dan **PUBLIC** (pengumuman), nyalakan *Enabled*, tekan **Send test**. Cara membuat webhook: Discord -> pengaturan channel -> Integrations -> Webhooks -> New Webhook -> Copy URL. URL disimpan **terenkripsi** dan tidak pernah ditampilkan lagi |
| **Products** | `SEED-PRODUCTS.bat` memuat 3 contoh dari folder `seed` (opsional), atau **+ CREATE NEW PRODUCT** untuk memasang script `.lua` Anda sendiri |
| **Users** | **Create account**: username Roblox pelanggan + (opsional) password. Bila dikosongkan, sistem membuat **password sementara yang tampil SEKALI**: salin dan kirim ke pelanggan lewat Discord. Pelanggan **wajib menggantinya** saat login pertama |
| **Licenses** | **Grant** lisensi produk ke pelanggan setelah pembayaran |
| **Trial Keys** | Buat key `ISID-XXXX-XXXX-XXXX` untuk dibagikan; terlihat di sini |

Pelanggan **tidak bisa mendaftar sendiri**: akun selalu dibuat admin. Di website tertulis "Belum punya akun? Hubungi atau masuk Discord Isekai Station ID".

## Langkah 10: Wallpaper dan logo (opsional)

Letakkan `wallpaper.png` dan `logo.gif` sesuai `docs\PANDUAN-TAMPILAN.md`, lalu `DEPLOY-NETLIFY.bat` lagi.

## Langkah 11: Teks Terms of Service (PENTING)

`site\public\assets\js\legal-content.js` berisi **draf** Terms of Service (8 bagian + Developer Statement), Privacy Policy, dan Contact Support yang saya tulis berdasarkan poin-poin Anda.
**Periksa dan sesuaikan dengan kebijakan Anda sendiri**; ini bukan nasihat hukum. Bila mengubahnya, naikkan tanggal versi di **dua tempat sekaligus**
(`server\src\lib\legal.js` -> `LEGAL_VERSION` dan `version` di `legal-content.js`; tes otomatis memastikan keduanya sama), jalankan ulang `START-ISID.bat` (restart backend) lalu `DEPLOY-NETLIFY.bat`. Semua pelanggan akan diminta menyetujui versi baru.

## Langkah 12: Jika ada masalah

| Gejala | Solusi |
|---|---|
| `[GAGAL] Node.js ...` | Pasang Node.js LTS 22.13+ lalu buka ulang jendela |
| `Cloudflared Service: GAGAL (service belum terpasang)` | Ulangi langkah 4 butir 3-4 (CMD **Administrator**) |
| `Hostname tunnel: PERINGATAN` | Cek Cloudflare: tunnel HEALTHY dan route menuju `127.0.0.1:8787`; hostname di `.env` sama dengan route (`SETUP-TUNNEL.bat`) |
| `PORT_BUSY` | Ada program lain memakai port 8787. Tutup program itu, atau ubah `PORT` di `.env` **dan** URL route di Cloudflare |
| Website tampil, login "Server maintenance" | Backend/service belum jalan: `START-ISID.bat` |
| 403 "Access denied" dari backend | `ISID_GATEWAY_SECRET` di `.env` tidak sama dengan di Netlify: jalankan `DEPLOY-NETLIFY.bat` |
| Lupa password admin | `RESET-ADMIN-PASSWORD.bat` |
| Deploy macet/gagal | Baca RINGKASAN; jalur manual ditampilkan skrip (pasang dua variabel di Netlify -> Enter) |

Selesai. Mulai sekarang: **`START-ISID.bat`** saja (lihat `PANDUAN-SETELAH-TERPASANG.md`). Jangan jalankan ulang `CREATE-ADMIN.bat`, pemasangan service Cloudflared, atau `SETUP.bat` tanpa alasan.

## Catatan jujur

Semua skrip Node dan tampilan diuji otomatis di Linux. **File `.bat`, Cloudflare, Netlify CLI, Roblox, executor, dan Discord asli belum pernah dijalankan oleh pembuat paket ini**; kerjakan `docs\UJI-MANUAL.md` sebelum berjualan.
