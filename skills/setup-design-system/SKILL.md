---
name: setup-design-system
description: Skill ini menetapkan identitas visual proyek sebagai token desain (warna, font, ukuran teks, spasi, radius, bayangan, gerak, tema terang/gelap) di satu sumber, mencatatnya di docs/DESIGN.md, dan memasang pengecekan supaya UI hanya memakai token. Pakai saat memulai tampilan proyek, mengganti tema/warna/font/identitas, menambah mode gelap, atau saat tampilan tidak konsisten karena warna dan ukuran ditulis sembarangan.
license: MIT
metadata:
  version: "0.5"
  language: id
  status: draf
---

# Sistem desain

## Tujuan

Semua keputusan visual ada di satu tempat. Komponen hanya memakai token, jadi seluruh web terlihat seragam,
mengganti warna/tema cukup di satu file, dan siapa pun yang menambah halaman otomatis mengikuti gaya yang sama.

## Kapan dipakai

- Sebelum halaman pertama proyek dibuat.
- Mengganti identitas, warna, font, atau menambah mode gelap.
- Tampilan tidak konsisten.

**Tidak dipakai untuk:** membuat komponen dan halaman (`build-ui`).

## Data proyek yang dibutuhkan

- Identitas dari user (lihat langkah 1). Logo/warna merek kalau sudah ada.
- `docs/DESIGN.md` kalau sudah ada: **ubah, jangan buat ulang**.

## Langkah

### 1. Tanya identitas

Prioritas audit: P4.

Satu pesan, bahasa sederhana, dengan rekomendasi:

1. Kesan yang diinginkan (pilih 2-3 kata): tegas, teknis, ramah, mewah, ceria, tenang, …
2. Warna merek/logo yang sudah ada? Kalau belum, usulkan satu warna aksen yang cocok dengan kesannya.
3. Mode terang, gelap, atau keduanya? (Rekomendasi: keduanya, mengikuti pengaturan perangkat.)
4. Preferensi font? (Rekomendasi: satu font sans-serif yang mudah dibaca, plus font monospace bila ada kode atau
   angka teknis.)

### 2. Susun token dalam dua lapis [DTCG, Material 3: Color roles]

Prioritas audit: P4.

**Lapis 1, primitif:** palet mentah. Contoh: `brand-50 … brand-950`, `neutral-50 … neutral-950`, dan palet
status. **Komponen tidak boleh memakai lapis ini langsung.**

**Lapis 2, semantik:** nama berdasarkan *fungsi*, menunjuk ke primitif. Ini satu-satunya yang dipakai
komponen.

| Token semantik | Dipakai untuk |
|---|---|
| `background` | latar halaman |
| `surface` | latar kartu, panel, dialog |
| `text` | teks utama |
| `text-muted` | teks sekunder, keterangan |
| `border` | garis hiasan dan pemisah |
| `border-strong` | batas input dan kontrol form (wajib ≥ 3 : 1) |
| `primary` / `primary-foreground` | tombol utama, tautan, aksen / teks di atasnya |
| `danger`, `success`, `warning`, `info` (+ `-foreground`) | status |
| `focus` | garis fokus keyboard |

Mode gelap = **nilai lapis 2 diganti**, nama tetap. Komponen tidak perlu berubah.

### 3. Batasi pilihan

Prioritas audit: P4.

| Jenis | Aturan |
|---|---|
| Warna | satu aksen merek + netral + 4 warna status |
| Font | maksimal 2 keluarga (teks + monospace opsional) |
| Ukuran teks | skala tetap, mis. 12, 14, 16, 18, 20, 24, 30, 36, 48 px; **teks bacaan** (paragraf, isi artikel/dokumen) minimal 16 px, tinggi baris ±1.5; label, tabel padat, dan keterangan boleh 12-14 px |
| Spasi | kelipatan 4 px |
| Radius | 2-3 nilai (mis. kecil untuk input, sedang untuk kartu, penuh untuk pil) |
| Bayangan | 2-3 tingkat |
| Gerak | 150-300 ms; dimatikan bila pengguna memilih `prefers-reduced-motion` |
| Breakpoint | sedikit, desain mulai dari layar HP (*mobile-first*) |
| Ikon | satu pustaka ikon untuk seluruh proyek |

Nilai di luar skala tidak dipakai. Butuh nilai baru → tambah token, bukan menulis angka langsung.

### 4. Kontras [WCAG 2.2]

Prioritas audit: P3.

Setiap pasangan teks/latar dan elemen UI dicek dengan alat pengukur kontras:

- teks biasa: minimal **4.5 : 1** (SC 1.4.3);
- teks besar (≥24 px, atau ≥18.66 px tebal): minimal **3 : 1**;
- batas komponen (input, tombol bergaris), ikon penting, garis fokus: minimal **3 : 1** (SC 1.4.11);
- informasi tidak disampaikan **hanya** dengan warna (SC 1.4.1): status juga punya teks atau ikon.

Cek di mode terang **dan** gelap. Catat rasionya di `docs/DESIGN.md`.

### 5. Pasang di kode

Prioritas audit: P4.

- Token ditulis **sekali** di satu file (cara per stack di `references/`).
- Matikan palet bawaan framework kalau bisa, supaya warna di luar token tidak tersedia sama sekali.
- Pasang pengecekan otomatis (lint/skrip di CI) yang menolak warna hex/rgb langsung dan nilai sembarang di
  komponen.

### 6. Catat

Prioritas audit: P4.

- Isi `docs/DESIGN.md` dari `assets/DESIGN.md`. Kalau proyek sudah punya dokumen desain dengan nama lain
  (mis. `docs/DESIGN-SYSTEM.md`), **jangan membuat file kedua**: lengkapi file itu dengan bagian yang kurang
  dari aset, dan catat namanya di *Konvensi* `docs/PROJECT.md`. Satu topik, satu dokumen (`write-docs`).
- Identitas pertama dan setiap penggantian identitas dicatat sebagai ADR (`build-feature`).
- Buat satu halaman contoh khusus pengembangan yang menampilkan semua token dan komponen dasar, supaya user
  bisa melihat dan menyetujui. Halaman ini tidak tampil di produksi.

## Larangan

- Warna, ukuran, atau jarak ditulis langsung di komponen (`#1a2b3c`, `13px`, `text-[15px]`).
- Komponen memakai token primitif (`brand-500`) alih-alih semantik (`primary`).
- Nama token berdasarkan tampilan (`blue`, `big`) di lapis semantik.
- Menambah warna/font baru tanpa mencatatnya di `docs/DESIGN.md`.

## Cek hasil

- [ ] `docs/DESIGN.md` terisi; semua pasangan warna punya rasio kontras yang lolos.
- [ ] Token ada di satu file; mode gelap hanya mengganti nilai.
- [ ] Pengecekan otomatis berjalan dan lulus.
- [ ] Halaman contoh token disetujui user.

## Aset

- `assets/DESIGN.md`: kerangka dokumen desain proyek.

## Stack

- `references/tailwind.md`: Tailwind CSS v4 (+ catatan v3)

## Sumber

- [Design Tokens Community Group: Format Module 2025.10](https://www.designtokens.org/technical-reports/): konsep token.
- [Material Design 3: Color roles](https://m3.material.io/styles/color/roles): token semantik berdasarkan fungsi.
- [WCAG 2.2](https://www.w3.org/TR/WCAG22/): SC 1.4.1, 1.4.3, 1.4.11.
- [MDN: prefers-reduced-motion](https://developer.mozilla.org/en-US/docs/Web/CSS/@media/prefers-reduced-motion).
- [MDN: prefers-color-scheme](https://developer.mozilla.org/en-US/docs/Web/CSS/@media/prefers-color-scheme).
