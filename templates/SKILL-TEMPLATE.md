---
name: nama-skill
description: Skill ini <apa yang dikerjakan, satu kalimat>. Pakai saat <situasi pemicu, dengan kata kunci yang mungkin diucapkan user>.
license: MIT
metadata:
  version: "0.1"
  language: id
  status: draf
---

<!--
Aturan menulis skill (hapus komentar ini di skill jadi):
- Nama folder = name. Huruf kecil, angka, tanda hubung. Pola kata kerja + objek.
- Umum: jangan sebut nama proyek, warna, layanan, atau path khas proyek tertentu.
- Satu topik. Kalau menyentuh topik skill lain, rujuk skill itu ("ikuti `design-api`"), jangan salin isinya.
- SKILL.md di bawah 500 baris. Detail per stack ke references/<stack>.md (satu tingkat saja).
- Setiap aturan wajib punya sumber di bagian Sumber.
- Bahasa sederhana; istilah teknis dijelaskan sekali saat pertama muncul.
- Wajib vs contoh: isi "Aturan wajib" dan "Larangan" adalah prinsip yang berlaku di semua proyek. Nama
  folder/file, format, atau alat tertentu ditulis di baris "Contoh bentuk:" supaya jelas boleh diganti.
  Kalau satu hal memang harus persis begitu (mis. algoritma hash), tulis di aturan, bukan di contoh.
- Setiap aturan bernomor punya baris "Prioritas audit" (dipakai `start-project` mode B).
-->

# Judul skill

## Tujuan

Apa hasil akhirnya kalau skill ini diikuti, dalam 2-3 kalimat.

## Kapan dipakai

- Situasi 1
- Situasi 2

**Tidak dipakai untuk:** situasi yang mirip tapi milik skill lain (sebut skill-nya).

## Data proyek yang dibutuhkan

Yang harus dibaca dari repo proyek sebelum mulai. Kalau tidak ada, **tanya user**, jangan mengarang.

- Contoh: stack proyek, file desain, ADR terkait

## Langkah

1. Langkah pertama
2. Langkah kedua

## Aturan wajib

### 1. Nama aturan [sumber]

Prioritas audit: P1/P2/P3/P4 (boleh per butir: "P1 untuk …; P3 lainnya").

- Butir aturan yang tidak boleh dilanggar.

Contoh bentuk: cara/nama yang sudah diuji; proyek boleh memakai bentuk lain yang memenuhi aturan di atas.

## Larangan

- Hal yang sering dilakukan AI/pemula tapi salah.

## Cek hasil

Checklist sebelum tugas dianggap selesai:

- [ ] Poin yang bisa dicek dengan jelas (ya/tidak)

## Stack

Cara spesifik per framework/alat ada di `references/`:

- `references/<stack>.md`

Stack yang belum ada filenya: beri tahu user, lihat `CATALOG.md`.

## Sumber

- [Nama sumber](url): bagian yang dipakai
