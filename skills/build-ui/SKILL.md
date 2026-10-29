---
name: build-ui
description: Skill ini membuat komponen, halaman, dan form yang konsisten dengan sistem desain - pemakaian ulang komponen dasar, varian lewat props, keadaan memuat/kosong/error, form yang benar, aksesibilitas (WCAG 2.2 AA), responsif, performa (Core Web Vitals), dan SEO. Pakai saat membuat atau mengubah komponen UI, halaman, form, tabel/daftar, dialog, atau saat tampilan berantakan, lambat, tidak rapi di HP, atau sulit dipakai dengan keyboard.
license: MIT
metadata:
  version: "0.4"
  language: id
  status: draf
---

# Membangun UI

## Tujuan

Setiap halaman dirakit dari komponen yang sama, terlihat seragam, bisa dipakai semua orang (termasuk lewat
keyboard dan pembaca layar), enak di HP, dan cepat dibuka.

## Kapan dipakai

- Membuat/mengubah komponen, halaman, form, daftar, dialog.
- Tampilan tidak konsisten, lambat, rusak di HP, sulit dipakai.

**Tidak dipakai untuk:** menentukan warna/font/token (`setup-design-system`), teks dan terjemahan
(`manage-translations`), pemanggil API (`design-api`).

## Data proyek yang dibutuhkan

- `docs/DESIGN.md` (token + daftar komponen). Belum ada → `setup-design-system` dulu.
- Letak folder dari `structure-project`.

## Aturan wajib

### 1. Komponen

- **Cek dulu** daftar komponen di dokumen desain (`docs/DESIGN.md` atau nama yang tercatat di *Konvensi*). Kalau sudah ada, pakai. Kalau hampir cocok, tambah varian.
  Tidak membuat komponen kembar.
- **Varian lewat props** (`variant`, `size`), dengan pilihan terbatas yang tercatat di `docs/DESIGN.md`.
  Bukan menyalin komponen lalu mengubah sedikit.
- **Komponen dasar** ada di folder UI bersama dan dibuat paling lambat saat pertama kali dibutuhkan, sebelum
  halaman yang memakainya. Contoh bentuk: Button, Input, Textarea, Select, Checkbox, Field (label + bantuan +
  error), Dialog, Toast, Table/List, Badge, Skeleton, EmptyState, ErrorState, Pagination.
- Elemen HTML interaktif mentah (`<button>`, `<input>`, `<select>`) **hanya** dipakai di dalam komponen dasar.
  Halaman dan fitur memakai komponen dasar. Di proyek lama, halaman yang masih memakai elemen mentah adalah
  *utang*: kode **baru** wajib memakai komponen dasar; halaman lama dipindah saat halaman itu disentuh atau
  lewat tugas rapi-rapi tersendiri. "Disentuh" berarti bagian yang diubah: menambah satu field di halaman lama
  cukup memakai komponen dasar untuk field itu, tanpa merombak seluruh halaman di tugas yang sama.
- **Hanya token semantik** untuk warna, ukuran, jarak (`setup-design-system`).
- Komponen dasar tidak mengambil data dan tidak berisi teks bisnis; teks datang lewat props.
- Komponen baru dasar dicatat di tabel komponen `docs/DESIGN.md`.

### 2. Halaman

Kerangka setiap halaman:

```
Judul halaman (satu h1) + aksi utama
Isi
```

Setiap bagian yang memuat data punya **empat keadaan**:

| Keadaan | Tampilan |
|---|---|
| Memuat | skeleton seukuran isi asli (layout tidak melompat) |
| Kosong | penjelasan singkat + aksi berikutnya ("Belum ada kunci API. Buat kunci") |
| Error | pesan yang dimengerti user + tombol coba lagi |
| Berhasil | isi |

- Aksi yang merusak (hapus, cabut, batalkan) minta konfirmasi lewat Dialog, bukan `confirm()` bawaan browser.
- Umpan balik aksi singkat → Toast. Error yang harus diperbaiki user → di tempatnya (dekat field / di bagian
  itu), bukan Toast yang hilang sendiri.

### 3. Form [WAI: Forms tutorial]

- Setiap field punya **label yang selalu terlihat**. Placeholder bukan label.
- Field wajib ditandai; format yang diharapkan dijelaskan sebelum user salah.
- Validasi saat kirim, lalu per field saat field ditinggalkan. Pesan error **di dekat field**, terhubung ke
  field (`aria-describedby`), dan fokus pindah ke error pertama.
- Error dari server (`errors` per field, `design-api`) ditampilkan di field yang sama.
- Tombol kirim nonaktif + indikator selama proses. Aksi uang/pesanan memakai `Idempotency-Key`.
- Isi form **tidak dihapus** saat terjadi error.
- Tipe input dan `autocomplete` yang tepat (`email`, `tel`, `current-password`, `one-time-code`).

### 4. Aksesibilitas, target WCAG 2.2 AA [WCAG 2.2]

- HTML semantik: `<button>` untuk aksi, `<a>` untuk pindah halaman, urutan judul berjenjang, landmark
  (`header`, `nav`, `main`, `footer`).
- Semua bisa dipakai **dengan keyboard saja**, urutan fokus masuk akal.
- Fokus selalu terlihat (SC 2.4.7) dan tidak tertutup elemen lain (SC 2.4.11).
- Gambar informatif punya `alt`; gambar hiasan `alt=""`.
- ARIA hanya kalau elemen HTML biasa tidak cukup. [WAI-ARIA APG: Read me first]
- Target sentuh minimal 24×24 px (SC 2.5.8); tombol utama di HP lebih baik ±44 px.
- Dialog: fokus masuk ke dialog, `Esc` menutup, fokus kembali ke pemicu.
- Bahasa halaman diset (`<html lang>`).

### 5. Responsif

- Rancang dari layar HP dulu, lalu lebar.
- Cek di lebar ±360, 768, 1280 px. Lebar 320 px tidak boleh memunculkan scroll horizontal di halaman
  (SC 1.4.10), dan halaman tetap terpakai saat di-zoom 200%.
- Tabel lebar: di HP jadi daftar kartu, atau scroll horizontal **di dalam** wadah tabel saja.

### 6. Performa [web.dev: Core Web Vitals]

Target pada persentil ke-75 pengunjung: **LCP ≤ 2.5 s, INP ≤ 200 ms, CLS ≤ 0.1**.

- Gambar: ukuran (lebar/tinggi) selalu ditentukan, format modern, `lazy` untuk yang di bawah layar, gambar
  utama di atas layar tidak di-lazy. Komponen gambar framework (mis. `next/image`) dipakai untuk foto dan
  gambar konten; ikon/logo kecil statis (SVG, bendera) dan gambar dari data (mis. QR yang dibuat di browser)
  boleh `<img>` biasa, asal lebar/tingginya tertulis.
- Font di-host sendiri dengan `font-display: swap`; jumlah ketebalan seperlunya.
- JavaScript di browser seminimal mungkin; komponen berat dimuat saat dibutuhkan.
- Ruang untuk isi yang dimuat belakangan disiapkan (skeleton, ukuran tetap) supaya tidak melompat.
- Daftar panjang dipecah per halaman.
- Library tambahan dicek dulu ukurannya dan lewat `secure-feature`.

### 7. SEO (halaman publik) [Google: SEO Starter Guide]

- `title` dan `description` unik per halaman; satu `h1`.
- URL bersih dan bermakna; `canonical` untuk halaman yang bisa diakses dari beberapa URL.
- `sitemap.xml`, `robots.txt`, gambar Open Graph.
- Halaman privat (dashboard, admin) diberi `noindex`.

## Larangan

- Menyalin komponen; membuat tombol/input versi sendiri di halaman.
- Warna/ukuran ditulis langsung.
- Halaman tanpa keadaan memuat/kosong/error.
- `div` atau `span` yang bisa diklik sebagai pengganti tombol/tautan.
- `alert()`/`confirm()` bawaan browser.
- Menghapus garis fokus (`outline: none`) tanpa pengganti.

## Cek hasil

- [ ] Tidak ada elemen interaktif mentah di luar komponen dasar; pengecekan desain lulus.
- [ ] Empat keadaan ada di setiap bagian yang memuat data.
- [ ] Dicoba dengan keyboard saja: semua aksi bisa, fokus terlihat.
- [ ] Dicek di lebar 360 dan 1280 px; tidak ada scroll horizontal halaman.
- [ ] Halaman publik: title, description, h1 tunggal; Lighthouse tidak menunjukkan masalah besar.
- [ ] Komponen baru tercatat di `docs/DESIGN.md`.

## Stack

- `references/nextjs.md`: Next.js App Router

## Sumber

- [WCAG 2.2](https://www.w3.org/TR/WCAG22/): SC 1.4.10, 2.4.7, 2.4.11, 2.5.8.
- [WAI: Forms tutorial](https://www.w3.org/WAI/tutorials/forms/).
- [WAI-ARIA Authoring Practices: Read me first](https://www.w3.org/WAI/ARIA/apg/practices/read-me-first/).
- [web.dev: Web Vitals](https://web.dev/articles/vitals).
- [web.dev: Optimize CLS](https://web.dev/articles/optimize-cls).
- [Google Search Central: SEO Starter Guide](https://developers.google.com/search/docs/fundamentals/seo-starter-guide).
