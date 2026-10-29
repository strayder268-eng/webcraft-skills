---
name: write-docs
description: Skill ini menulis dan merawat dokumentasi yang selalu cocok dengan kode - README, ringkasan proyek, dokumentasi API (OpenAPI), changelog, panduan untuk pengguna, dan runbook - dengan pembagian jenis dokumen Diátaxis, bahasa sederhana, contoh yang benar-benar dijalankan, dan diperbarui di commit yang sama dengan kodenya. Pakai saat membuat atau mengubah README, menambah/mengubah endpoint API, merilis perubahan, menulis panduan pemakaian, atau saat dokumentasi sudah tidak sesuai kode.
license: MIT
metadata:
  version: "0.5"
  language: id
  status: draf
---

# Menulis dokumentasi

## Tujuan

Orang baru (atau agent lain) bisa menjalankan proyek dari README tanpa bertanya, pemakai API bisa memakai API
dari dokumentasinya saja, dan tidak ada dokumen yang menyesatkan karena usang.

## Kapan dipakai

README, dokumentasi API, changelog, panduan pengguna, runbook, dokumen yang usang. Dipanggil dari
`build-feature` setiap kali perubahan memengaruhi dokumen.

## Aturan wajib

### 1. Empat jenis dokumen [Diátaxis]

Prioritas audit: P4.

Satu dokumen untuk satu kebutuhan; jangan dicampur.

| Jenis | Pembaca butuh | Contoh |
|---|---|---|
| **Tutorial** | belajar dari nol, langkah demi langkah | "Membuat kunci API pertamamu" |
| **Panduan (how-to)** | menyelesaikan tugas tertentu | "Cara top-up lewat QRIS", "Cara memulihkan backup" |
| **Referensi** | mencari fakta persis | dokumentasi endpoint API, daftar env |
| **Penjelasan** | memahami alasan | ADR, "kenapa saldo memakai ledger" |

### 2. Letak dokumen

Prioritas audit: P4.

| Dokumen | Letak | Isi |
|---|---|---|
| README | `README.md` | apa ini, untuk siapa, cara menjalankan lokal (satu perintah), cara test, tautan ke dokumen lain, lisensi |
| Ringkasan proyek | `docs/PROJECT.md` | lihat `start-project` |
| Desain | `docs/DESIGN.md` (atau nama yang tercatat di *Konvensi*) | lihat `setup-design-system` |
| Fitur dan keputusan | `docs/features/`, `docs/decisions/` | lihat `build-feature` |
| API | `docs/api/openapi.yaml` (+ halaman panduan) | aturan 4 |
| Runbook | `docs/runbooks/` | lihat `operate-app` |
| Perubahan | `CHANGELOG.md` | aturan 5 |
| Bantuan pengguna | halaman bantuan di aplikasi | panduan per tugas, bahasa awam |

Informasi ditulis **di satu tempat**; tempat lain menautkan, tidak menyalin.

### 3. Selalu cocok dengan kode

Prioritas audit: P3; dokumen yang menyuruh hal berbahaya: P1.

- Dokumen diperbarui **di commit yang sama** dengan perubahan kodenya.
- Setiap perintah dan contoh di dokumen **sudah dijalankan** dan berhasil. README dicoba dari clone bersih.
- Dokumen usang lebih berbahaya daripada tidak ada: perbarui atau hapus, jangan dibiarkan. Saat audit atau
  saat fitur/halaman dihapus, periksa **semua** file di `docs/` (bukan hanya yang namanya cocok): dokumen yang
  masih menyebut fitur, halaman, atau layanan yang sudah tidak ada diperbarui, atau diberi tanda **ARSIP** di
  baris pertama dan dipindah ke `docs/archive/`. Menghapus dokumen butuh izin user. Saat audit
  (`start-project` mode B) dokumen usang hanya dicatat; pengarsipan dikerjakan di tugas berikutnya.
- Tanggal ditulis lengkap (2026-10-29), bukan "kemarin" atau "minggu depan".
- Tidak ada rahasia: pakai contoh (`sk-…`, `<TOKEN>`), dan env cukup disebut namanya.

### 4. Dokumentasi API [OpenAPI 3.1]

Prioritas audit: P3.

- API untuk pihak luar didokumentasikan dalam file OpenAPI: setiap endpoint, cara autentikasi, skema
  request/response, **daftar `code` error**, contoh sukses dan gagal.
- API yang mengikuti kontrak pihak lain (`design-api` aturan 2): cukup halaman yang merujuk kontrak itu
  (versi/tautan) ditambah daftar **perbedaan dan tambahan** (endpoint, field, `code` error milik proyek).
  OpenAPI wajib untuk endpoint tambahan itu saja.
- File OpenAPI divalidasi di CI dan dicocokkan dengan perilaku API (test kontrak atau generate dari kode).
- Ada halaman "Mulai cepat" dengan satu contoh `curl` lengkap yang bisa langsung dicoba.
- Perubahan yang merusak kompatibilitas diumumkan di changelog dengan cara migrasinya (`design-api`
  aturan 12).

### 5. Changelog [Keep a Changelog]

Prioritas audit: P4.

- Untuk manusia, bukan salinan log git.
- Bagian `Unreleased` di atas; setiap versi bertanggal.
- Kelompok: `Added`, `Changed`, `Deprecated`, `Removed`, `Fixed`, `Security` (boleh diterjemahkan, konsisten).
- Yang dicatat: perubahan yang terasa oleh pemakai/pemakai API.

### 6. Gaya tulisan [Google developer documentation style guide]

Prioritas audit: P4.

- Kalimat pendek, bentuk aktif, langsung ke pembaca ("Jalankan …", bukan "Perintah berikut dapat dijalankan").
- Istilah teknis dijelaskan sekali saat pertama muncul; pakai istilah yang sama di seluruh dokumen.
- Langkah bernomor untuk prosedur; tabel untuk perbandingan; blok kode diberi bahasa (`sh`, `json`).
- Bantuan pengguna memakai bahasa awam dan berangkat dari tugas pengguna, bukan dari struktur menu.
- Gambar/screenshot seperlunya; cepat usang.

## Larangan

- Mengubah kode yang memengaruhi dokumen tanpa memperbarui dokumennya.
- Contoh perintah yang belum pernah dijalankan.
- Menyalin isi yang sama ke banyak dokumen.
- Rahasia atau data user di dokumen/screenshot.

## Cek hasil

- [ ] README dicoba dari clone bersih: proyek jalan dengan langkah yang tertulis.
- [ ] Endpoint baru/berubah tercermin di OpenAPI; validasi lulus.
- [ ] `CHANGELOG.md` diperbarui untuk perubahan yang terasa pemakai.
- [ ] Tautan di dokumen yang diubah tidak rusak.

## Aset

- `assets/README.md`: kerangka README

## Sumber

- [Diátaxis](https://diataxis.fr/).
- [Google developer documentation style guide](https://developers.google.com/style).
- [Keep a Changelog 1.1.0](https://keepachangelog.com/en/1.1.0/).
- [OpenAPI Specification 3.1](https://spec.openapis.org/oas/v3.1.0).
- [Write the Docs: Docs as Code](https://www.writethedocs.org/guide/docs-as-code/).
