---
name: manage-translations
description: Skill ini mengatur semua teks yang tampil ke user lewat kamus terjemahan - struktur file per bahasa, nama kunci, placeholder dan bentuk jamak, format angka/uang/tanggal per bahasa, bahasa di URL dan SEO, terjemahan kode error API, serta pengecekan kunci yang hilang. Pakai saat menambah atau mengubah teks UI, email, atau notifikasi, menambah bahasa baru, menampilkan uang/tanggal/angka, atau saat ada teks yang belum diterjemahkan.
license: MIT
metadata:
  version: "0.5"
  language: id
  status: draf
---

# Mengelola terjemahan

## Tujuan

Tidak ada teks yang ditulis langsung di kode. Menambah bahasa baru cukup dengan **menambah satu folder kamus
dan satu baris di daftar bahasa**, dan pengecekan otomatis memberi tahu kalau ada teks yang terlewat.

## Kapan dipakai

- Menambah/mengubah teks yang dilihat user (UI, email, notifikasi, pesan error).
- Menambah bahasa.
- Menampilkan angka, uang, tanggal, waktu.

**Tidak dipakai untuk:** isi konten dari database yang diisi user/admin (itu data, bukan teks UI).

## Data proyek yang dibutuhkan

- Daftar bahasa dan bahasa bawaan dari `docs/PROJECT.md`.

## Aturan wajib

### 1. Semua teks lewat kamus

Prioritas audit: P3.

Termasuk teks yang tidak terlihat langsung: `aria-label`, `alt`, `title`, `placeholder`, pesan error, judul
halaman, isi email dan notifikasi.

Pengecualian: nama merek/produk, data milik user, kode teknis yang memang ditampilkan apa adanya.

### 2. Struktur file

Prioritas audit: P4.

```
messages/
  id/
    common.json        teks umum: tombol, navigasi, error umum
    billing.json       satu file per fitur
  en/
    common.json
    billing.json
```

- Satu folder per bahasa, isi file sama persis di setiap bahasa.
- Daftar bahasa yang didukung ditulis **di satu tempat** di kode, bersama bahasa bawaan.
- Menambah bahasa = salin folder bahasa bawaan → terjemahkan → tambahkan ke daftar bahasa. Tidak ada kode lain
  yang perlu diubah.

Contoh bentuk: file JSON per bahasa per fitur seperti di atas. Bentuk lain sah kalau tetap memenuhi tiga poin
di atas, mis. kamus TypeScript bertipe (`dict/<fitur>.ts` yang memaksa kunci setiap bahasa sama dengan bahasa
bawaan). Yang **bukan** konvensi sah: daftar bahasa ditulis di lebih dari satu tempat (`'en' | 'id'` di dua file atau
lebih), karena menambah bahasa jadi harus mengubah banyak kode.

### 3. Nama kunci

Prioritas audit: P4.

- Berdasarkan **makna**, bukan isi teks: `billing.topup.submit`, bukan `"Top up sekarang"`.
- Dikelompokkan per fitur, lalu per layar/bagian.
- Kode error API diterjemahkan di `common.errors.<code>` (`design-api`).

### 4. Placeholder dan jamak [ICU MessageFormat, CLDR]

Prioritas audit: P3.

- **Jangan menyambung potongan teks.** Urutan kata berbeda di tiap bahasa.
  - Salah: `t('hello') + ', ' + name`
  - Benar: `"greeting": "Halo, {name}"`
- Jumlah memakai bentuk jamak sesuai aturan bahasa:
  `"{count, plural, one {# kunci} other {# kunci}}"` (Indonesia) dan
  `"{count, plural, one {# key} other {# keys}}"` (Inggris). Setiap bahasa punya kategori jamak sendiri
  (CLDR); jangan menulis `if (count === 1)` di kode.

### 5. Angka, uang, tanggal [MDN: Intl]

Prioritas audit: P3.

- Selalu diformat dengan `Intl` sesuai bahasa user, tidak manual:
  - `new Intl.NumberFormat('id-ID', { style: 'currency', currency: 'IDR', maximumFractionDigits: 0 })`
    → `Rp 150.000`
  - `new Intl.DateTimeFormat('id-ID', { dateStyle: 'medium', timeStyle: 'short', timeZone })`
- Waktu disimpan UTC (`change-database`), **diubah ke zona waktu user hanya saat ditampilkan**.
- Uang dari API dalam satuan terkecil; dikonversi sesuai jumlah desimal mata uangnya sebelum diformat.

### 6. Bahasa di URL dan SEO [Google: Localized versions]

Prioritas audit: P3.

- Halaman publik memakai awalan bahasa di URL (`/id/…`, `/en/…`), supaya setiap bahasa bisa diindeks.
- Setiap halaman publik mencantumkan versi bahasa lainnya (`hreflang`) termasuk `x-default`.
- `<html lang>` sesuai bahasa halaman. [W3C: Declaring language]
- Kunjungan pertama tanpa awalan boleh diarahkan sesuai `Accept-Language`; setelah user memilih bahasa,
  pilihannya disimpan (cookie/profil) dan dihormati. Tidak memaksa bahasa berdasarkan lokasi IP.

### 7. Teks dari backend

Prioritas audit: P3.

Email dan notifikasi memakai kamus yang sama polanya, dengan bahasa yang tersimpan di profil user.

### 8. Tata letak tahan terjemahan

Prioritas audit: P3.

- Teks di bahasa lain bisa ±30% lebih panjang: jangan memberi lebar tetap pada elemen berisi teks.
- Pakai properti CSS logis (`margin-inline-start`, bukan `margin-left`; di Tailwind `ms-`/`me-`/`ps-`/`pe-`)
  supaya siap untuk bahasa kanan-ke-kiri. [MDN: CSS logical properties]

### 9. Pengecekan otomatis (CI)

Prioritas audit: P4.

- Setiap bahasa punya kunci yang sama persis dengan bahasa bawaan.
- Placeholder di setiap terjemahan sama dengan bahasa bawaan.
- Kunci yang hilang saat aplikasi berjalan: tampilkan teks bahasa bawaan dan catat peringatan di mode
  pengembangan.

### 10. Kualitas terjemahan

Prioritas audit: P3.

Terjemahan mesin/AI untuk halaman publik ditandai dan dibaca ulang oleh penutur bahasa itu sebelum dianggap
selesai.

## Larangan

- Teks ditulis langsung di komponen.
- Menyambung potongan kalimat.
- `if (count === 1)` untuk jamak.
- Format uang/tanggal manual (`'Rp ' + n`).
- Kunci berdasarkan isi teks.

## Cek hasil

- [ ] Tidak ada teks user-facing yang ditulis langsung di kode baru.
- [ ] Pengecekan kunci antar bahasa lulus.
- [ ] Angka, uang, tanggal memakai `Intl`.
- [ ] Halaman dicek di semua bahasa: tidak ada teks terpotong atau tata letak rusak.

## Stack

- `references/nextjs.md`: Next.js App Router

## Sumber

- [ICU MessageFormat](https://unicode-org.github.io/icu/userguide/format_parse/messages/).
- [Unicode CLDR: Plural rules](https://cldr.unicode.org/index/cldr-spec/plural-rules).
- [MDN: Intl](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl).
- [Google Search Central: Localized versions of your pages](https://developers.google.com/search/docs/specialty/international/localized-versions).
- [W3C: Declaring language in HTML](https://www.w3.org/International/questions/qa-html-language-declarations).
- [MDN: CSS logical properties](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_logical_properties_and_values).
