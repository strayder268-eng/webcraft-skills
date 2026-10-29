---
name: write-tests
description: Skill ini menulis test yang benar-benar menjaga perilaku - pembagian unit, integrasi, dan end-to-end, apa yang wajib diuji (alur utama, error, izin, uang, bersamaan), test yang stabil dan tidak bergantung waktu/jaringan, data test yang bersih, penanganan test tidak stabil, dan menjalankannya di CI. Pakai saat menambah fitur, memperbaiki bug, mengubah kode yang belum punya test, menulis test database/API/UI/browser, atau saat test gagal acak.
license: MIT
metadata:
  version: "0.1"
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

| Tingkat | Menguji | Jumlah | Kecepatan |
|---|---|---|---|
| Unit | fungsi/aturan bisnis, tanpa jaringan dan database | banyak | milidetik |
| Integrasi | handler + service + database/Redis **asli**; kontrak API | sedang | detik |
| End-to-end | alur penting di browser sungguhan | sedikit | menit |

End-to-end hanya untuk alur yang kalau rusak langsung merugikan: daftar/login, pembayaran/top-up, fitur
utama produk.

### 2. Yang wajib diuji

Untuk setiap fitur:

- alur utama berhasil;
- input tidak valid (422) dan batasnya (nol, negatif, terlalu panjang, kosong);
- tanpa login (401) dan **milik orang lain** (404/403);
- kegagalan layanan luar (timeout, error) ditangani;
- uang/saldo/kuota: request **bersamaan** dan request **diulang** (idempotency) tidak membuat hitungan salah.

Untuk setiap bug: test yang mereproduksi bug, ditulis sebelum perbaikan (`fix-bug`).

### 3. Uji perilaku, bukan cara kerja dalam

- Test memeriksa hasil yang terlihat dari luar (response, data tersimpan, teks di layar), bukan fungsi privat
  atau urutan pemanggilan internal.
- UI diuji seperti user memakainya: cari elemen lewat peran dan label ("tombol Simpan"), bukan class CSS.
  [Testing Library: Guiding principles]
- Nama test menjelaskan perilaku: `menolak top-up dengan nominal negatif`.
- Pola isi: siapkan → jalankan → periksa.

### 4. Stabil (deterministik)

- **Tidak ada `sleep`** untuk menunggu. Tunggu kondisi yang jelas, atau kendalikan waktu (jam palsu yang
  disuntikkan).
- Tidak memanggil layanan luar sungguhan. Ganti dengan server tiruan lokal yang menjawab sesuai dokumentasi
  layanan itu (termasuk jawaban error).
- Database dan Redis **tidak** dipalsukan; pakai yang asli di container test. Yang dipalsukan hanya yang bukan
  milik kita.
- Test tidak bergantung urutan dan tidak saling bergantung.
- Angka acak memakai seed tetap atau tidak memengaruhi hasil.

### 5. Data test

- Database test **terpisah** (container sendiri). Tidak pernah menjalankan test ke database pengembangan yang
  berisi data, apalagi produksi.
- Setiap test membuat datanya sendiri dan **membersihkannya** (rollback transaksi, atau hapus di akhir test).
- Test yang berjalan paralel memakai data unik (ID/email unik per test) supaya tidak saling mengganggu.
- Cek kebocoran data: jumlah baris tabel utama sebelum dan sesudah seluruh test harus sama.

### 6. Test tidak stabil

Test yang kadang lulus kadang gagal adalah bug. Cari penyebabnya (waktu, urutan, data bersama, race), perbaiki.
Kalau belum bisa, karantina dengan catatan dan batas waktu perbaikan. Tidak dihapus diam-diam dan tidak
diakali dengan menaikkan timeout.

### 7. CI

- Semua test berjalan di setiap push/PR, bersama typecheck dan lint (`deploy-app`).
- Test gagal = tidak boleh di-merge/deploy.
- Coverage dipakai untuk mencari bagian yang belum teruji, bukan angka target.

## Larangan

- Menghapus/melewati test yang gagal supaya CI hijau.
- Mengubah ekspektasi test agar cocok dengan kode yang salah.
- Mock database sendiri.
- Test yang memanggil API pihak ketiga sungguhan.
- Data test tertinggal di database.

## Cek hasil

- [ ] Kasus wajib di aturan 2 tercakup untuk fitur/bug ini.
- [ ] Seluruh test lulus dua kali berturut-turut (cek stabil); untuk Go juga dengan `-race`.
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
