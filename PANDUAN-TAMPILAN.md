# Panduan tampilan: wallpaper, logo, warna

Tema ISID Hub v1.0.0: **gradasi hitam (atas) menuju magenta `#FF00FF` (bawah)** dengan panel kaca cair ala iOS.
Wallpaper dan logo animasi **tidak ikut di dalam ZIP**: Anda yang menaruhnya sendiri (dua file, nama harus persis). Tanpa kedua file itu website tetap tampil normal.

## 1. Wallpaper "ISEKAI STATION"

| | |
|---|---|
| Nama file (WAJIB persis) | `wallpaper.png` |
| Lokasi | `site\public\assets\img\wallpaper.png` (di folder `ISID-Hub`) |
| Ukuran disarankan | **1920 x 1080 px atau lebih** (landscape 16:9). Lebih kecil dari 1280 px akan tampak pecah di layar besar |
| Berat file | **di bawah ~1,5 MB** (dimuat tiap pengunjung; kecilkan di squoosh.app atau tinypng.com bila lebih besar) |
| Cara tampil | Ditarik memenuhi layar (`cover`, bagian pinggir bisa terpotong), **diam di tempat** saat halaman di-scroll, dicampur dengan mode *screen* pada kepekatan 38% di atas gradasi, jadi bagian hitam gambar "melebur" ke gradasi dan bagian terang/pink tetap terlihat |

Cara memasang:
1. Ganti nama file gambar Anda (mis. `file_000000006afc7207afa65e7a86dc85ca.png`) menjadi **`wallpaper.png`**.
   (Bila nama tidak tampak berakhiran `.png`, di File Explorer buka **View -> Show -> File name extensions** dulu, supaya tidak menjadi `wallpaper.png.png`.)
2. Salin ke `ISID-Hub\site\public\assets\img\`.
3. Klik dua kali **`DEPLOY-NETLIFY.bat`** supaya ikut terpasang di website. Lalu tekan **Ctrl+F5** di browser.

Terlalu terang/terlalu gelap? Buka `site\public\assets\css\isid.css` dengan Notepad, cari `--wallpaper-opacity: .38;` lalu ubah angkanya (`0` = hilang, `.2` = samar, `.6` = jelas). Deploy ulang.

## 2. Logo animasi "iS" (GIF)

| | |
|---|---|
| Nama file (WAJIB persis) | `logo.gif` |
| Lokasi | `site\public\assets\img\logo.gif` |
| Ukuran disarankan | **persegi** (rasio 1:1). Yang Anda punya, 480 x 480 px, sudah pas. Boleh lebih kecil (min. 128 x 128) |
| Berat file | **di bawah ~500 KB** (GIF animasi cepat membengkak; optimalkan di ezgif.com bila perlu) |
| Cara tampil | Menggantikan kotak "iS" kecil di pojok kiri atas (landing, halaman produk, dashboard pelanggan, dashboard admin). Ditampilkan sekitar **30 x 30 px** dengan sudut membulat dan dipotong persegi (`cover`), jadi **taruh bagian penting di tengah gambar** dan hindari tulisan kecil |

Cara memasang: ganti nama `ezgif.com-optimize_3.gif` menjadi **`logo.gif`**, salin ke `site\public\assets\img\`, jalankan `DEPLOY-NETLIFY.bat`.

## 3. Bila tidak muncul

| Gejala | Penyebab |
|---|---|
| Gambar tidak berubah | Belum menjalankan `DEPLOY-NETLIFY.bat`, atau browser menyimpan versi lama: tekan Ctrl+F5 (cache aset di server hanya 1 jam) |
| Tetap kotak "iS" | Nama/lokasi file salah (huruf kecil semua, `logo.gif` bukan `Logo.GIF`, bukan di subfolder lain) |
| Di Console browser (F12) ada galat 404 `/assets/img/logo.gif` atau `wallpaper.png` | Normal **selama file belum Anda taruh**; tidak merusak halaman |
| Teks sulit dibaca di atas wallpaper | Turunkan `--wallpaper-opacity` (bagian 1) |

Gambar dilayani sebagai file statis publik: **jangan** menaruh sesuatu yang rahasia di folder `assets\img`.

## 4. Mengubah warna

Semua warna berada di blok `:root { ... }` di `site\public\assets\css\isid.css` (bagian "TEMA"):
`--magenta` / `--violet` (warna aksen), `--text`, `--muted` (teks sekunder; jangan dibuat lebih gelap agar tetap terbaca di atas magenta),
dan gradasi latar pada `body::before` (hitam di atas -> `#ff00ff` di bawah). Setelah mengubah, deploy ulang.
Mengubah warna tidak memengaruhi fungsi apa pun.

## 5. Bahasa

Halaman pelanggan memakai **Indonesia (utama)** dan **English (kedua)**; pengunjung memilih lewat tombol **ID | EN** (pilihan diingat di browser). Dashboard admin sengaja tetap berbahasa Inggris.
