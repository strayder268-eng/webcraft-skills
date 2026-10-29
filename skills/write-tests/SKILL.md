---
name: write-tests
description: Skill ini menulis test yang benar-benar menjaga perilaku - pembagian unit, integrasi, dan end-to-end, apa yang wajib diuji (alur utama, error, izin, uang, bersamaan), test yang stabil dan tidak bergantung waktu/jaringan, data test yang bersih, penanganan test tidak stabil, dan menjalankannya di CI. Pakai saat menambah fitur, memperbaiki bug, mengubah kode yang belum punya test, menulis test database/API/UI/browser, atau saat test gagal acak.
license: MIT
metadata:
  version: "0.8"
  language: id
  status: draf
---

# Menulis test

## Tujuan

Test gagal kalau perilaku penting rusak, lulus kalau tidak, hasilnya sama setiap kali dijalankan, dan tidak
meninggalkan sampah data.

## Kapan dipakai

Setiap fitur baru (`build-feature`), setiap bug (`fix-bug`), sebelum merapikan kode, saat test tidak stabil.

## Data proyek yang dibutuhkan

- Perintah test dan cara menyiapkan database/Redis test (`docs/PROJECT.md`, `AGENTS.md`).

## Aturan wajib

### 1. Tiga tingkat [Fowler: Practical Test Pyramid]

Prioritas audit: P4.

| Tingkat | Menguji | Jumlah | Kecepatan |
|---|---|---|---|
| Unit | fungsi/aturan bisnis, tanpa jaringan dan database | banyak | milidetik |
| Integrasi | handler + service + database/Redis **asli**; kontrak API | sedang | detik |
| End-to-end | alur penting di browser sungguhan | sedikit | menit |

End-to-end hanya untuk alur yang kalau rusak langsung merugikan: daftar/login, pembayaran/top-up, fitur
utama produk.

Contoh bentuk: alat di `references/` (Vitest, Playwright, `go test`). Proyek yang sudah memakai test runner
lain (mis. `node --test`, Jest) tetap memakainya sebagai *konvensi*; yang wajib adalah tiga tingkat ini dan
kasus di aturan 2. Proyek yang belum punya test end-to-end sama sekali mencatatnya sebagai *utang*; menambah
alatnya butuh ADR (`CATALOG.md`).

### 2. Yang wajib diuji

Prioritas audit: P2 kalau uang, saldo, atau izin tidak punya test; P4 lainnya.

Untuk setiap fitur:

- alur utama berhasil;
- input tidak valid (422) dan batasnya (nol, negatif, terlalu panjang, kosong);
- tanpa login (401) dan **milik orang lain** (404/403);
- kegagalan layanan luar (timeout, error) ditangani;
- **input berbahaya** di setiap parameter teks yang sampai ke query (isian, pencarian, filter, urutan):
  `' OR '1'='1`, `'; DROP TABLE users; --`, `%`, `_`, `\`. Hasil yang benar: 422 untuk nilai yang tidak
  diizinkan, atau diperlakukan sebagai teks biasa. Setelah test, tabel dan jumlah barisnya tidak berubah;
- **urutan/filter di luar daftar yang diizinkan** ditolak 422, bukan diabaikan diam-diam;
- **jumlah query** endpoint daftar/halaman utama: tetap sama untuk 5 baris maupun 50 baris, dan tidak melebihi
  target di `build-backend` aturan 12. Test ini mencegah N+1 kembali diam-diam;
- uang/saldo/kuota: request **bersamaan** dan request **diulang** (idempotency) tidak membuat hitungan salah;
- **penjaga di rute** (login wajib, peran, step-up, rate limit): diuji lewat pemasangan rute yang **sama**
  dengan produksi (satu fungsi pendaftar rute yang dipakai `main` dan test), bukan router salinan di test.
  Request tanpa izin harus ditolak dan handler-nya **tidak dipanggil**. Contoh tanpa database di
  `references/go.md`.

Untuk setiap bug: test yang mereproduksi bug, ditulis sebelum perbaikan (`fix-bug`).

### 3. Uji perilaku, bukan cara kerja dalam

Prioritas audit: P4.

- Test memeriksa hasil yang terlihat dari luar (response, data tersimpan, teks di layar), bukan fungsi privat
  atau urutan pemanggilan internal.
- UI diuji seperti user memakainya: cari elemen lewat peran dan label ("tombol Simpan"), bukan class CSS.
  [Testing Library: Guiding principles]
- Nama test menjelaskan perilaku: `menolak top-up dengan nominal negatif`.
- Pola isi: siapkan → jalankan → periksa.

### 4. Stabil (deterministik)

Prioritas audit: P4.

- **Tidak ada `sleep`** untuk menunggu. Tunggu kondisi yang jelas, atau kendalikan waktu (jam palsu yang
  disuntikkan).
- Tidak memanggil layanan luar sungguhan. Ganti dengan server tiruan lokal yang menjawab sesuai dokumentasi
  layanan itu (termasuk jawaban error).
- Database dan Redis **tidak** dipalsukan; pakai yang asli di container test. Yang dipalsukan hanya yang bukan
  milik kita.
- Test tidak bergantung urutan dan tidak saling bergantung.
- Angka acak memakai seed tetap atau tidak memengaruhi hasil.

### 5. Data test

Prioritas audit: P2 kalau test bisa berjalan ke database berisi data asli; P4 lainnya.

- Database test **terpisah** (container sendiri). Tidak pernah menjalankan test ke database pengembangan yang
  berisi data, apalagi produksi.
- Proyek menyediakan **satu perintah** yang membuat database/Redis test sekali pakai, memuat skema, menjalankan
  test, menghitung sisa data, lalu menghapus container (dicatat di `AGENTS.md`). Agent boleh menjalankan perintah
  itu; dengan begitu test database bisa dijalankan sendiri sebelum melapor, bukan hanya di-skip. Contoh di
  `references/go.md`.
- Setiap test membuat datanya sendiri dan **membersihkannya** (rollback transaksi, atau hapus di akhir test).
- Test yang berjalan paralel memakai data unik (ID/email unik per test) supaya tidak saling mengganggu.
- Cek kebocoran data: jumlah baris tabel utama sebelum dan sesudah seluruh test harus sama.

### 6. Test tidak stabil

Prioritas audit: P4.

Test yang kadang lulus kadang gagal adalah bug. Cari penyebabnya (waktu, urutan, data bersama, race), perbaiki.
Kalau belum bisa, karantina dengan catatan dan batas waktu perbaikan. Tidak dihapus diam-diam dan tidak
diakali dengan menaikkan timeout.

### 7. CI

Prioritas audit: P3.

- Semua test berjalan di setiap push/PR, bersama typecheck dan lint (`deploy-app`).
- Test gagal = tidak boleh di-merge/deploy.
- Coverage dipakai untuk mencari bagian yang belum teruji, bukan angka target.

## Larangan

- Menghapus/melewati test yang gagal supaya CI hijau.
- Mengubah ekspektasi test agar cocok dengan kode yang salah.
- Mock database sendiri.
- Test yang memanggil API pihak ketiga sungguhan.
- Data test tertinggal di database.
- Test yang **membaca file kode sumber** lalu mencocokkan teks (`strings.Contains(source, "go func")`). Test
  seperti itu menguji cara menulis, bukan perilaku, dan rusak saat kode dirapikan. Ganti dengan test perilaku.
  Pemeriksaan isi file yang memang statis (Dockerfile punya `USER`, workflow CI dikunci SHA, skema memuat
  constraint) ditaruh sebagai skrip cek atau lint di CI, atau dibuktikan dengan memuat skema ke database
  test. Test lama yang sudah begitu dicatat sebagai *utang*.

## Cek hasil

- [ ] Kasus wajib di aturan 2 tercakup untuk fitur/bug ini.
- [ ] Seluruh test lulus dua kali berturut-turut (cek stabil); untuk Go juga dengan `-race`.
- [ ] Test yang dilewati (skip) **tidak** dihitung lulus. Laporan memisahkan lulus, gagal, dan skip beserta
  alasannya. Test yang di-skip karena butuh database/Redis wajib dijalankan di container test sebelum
  merge, oleh agent atau orang yang punya akses. Sebelum itu, status tugas **belum selesai**: laporkan
  "kode selesai, test database belum dijalankan".
- [ ] Tidak ada data tersisa setelah test.
- [ ] Perintah dan hasil test dilampirkan di laporan.

## Stack

- `references/go.md`: test tabel, httptest, database test, `-race`
- `references/nextjs.md`: Vitest + Testing Library, Playwright

## Sumber

- [Martin Fowler: The Practical Test Pyramid](https://martinfowler.com/articles/practical-test-pyramid.html).
- [Testing Library: Guiding principles](https://testing-library.com/docs/guiding-principles).
- [Playwright: Best practices](https://playwright.dev/docs/best-practices).
- [Go: testing](https://pkg.go.dev/testing).
