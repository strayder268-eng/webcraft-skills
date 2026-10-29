---
name: fix-bug
description: Skill ini memperbaiki bug sampai ke akar masalahnya (reproduksi, test yang gagal dulu, telusuri per lapisan, perbaikan minimal, cek dampak) dan merapikan kode tanpa mengubah perilaku (refactor). Pakai saat ada error, fitur tidak jalan, hasil salah, data tidak konsisten, test gagal atau tidak stabil, atau saat kode perlu dipecah/dirapikan tanpa menambah fitur.
license: MIT
metadata:
  version: "0.6"
  language: id
  status: draf
---

# Memperbaiki bug dan merapikan kode

## Tujuan

Bug diperbaiki di penyebabnya, bukan ditutupi gejalanya, dan dijaga test supaya tidak kembali. Kode dirapikan
tanpa ada perilaku yang ikut berubah.

## Kapan dipakai

- **Mode bug:** error, tidak jalan, hasil salah, data aneh, test gagal/tidak stabil.
- **Mode rapikan:** kode perlu dipecah, dipindah, diganti nama, atau disederhanakan tanpa mengubah perilaku.

**Tidak dipakai untuk:** menambah atau mengubah perilaku yang disengaja (`build-feature`), server mati atau
gangguan produksi (`operate-app` dulu, lalu kembali ke sini untuk perbaikan kodenya).

## Data proyek yang dibutuhkan

- Cara menjalankan dan menguji proyek (`docs/PROJECT.md`, `AGENTS.md`).
- Dari user: apa yang diharapkan, apa yang terjadi, langkah untuk memunculkan, sejak kapan. **Jangan minta**
  user mengirim sandi, token, atau isi `.env`.

## Mode bug

### 1. Pahami gejalanya

Tulis dalam satu-dua kalimat: "Diharapkan X, yang terjadi Y, saat melakukan Z." Baca pesan error dan log
**lengkap**, jangan hanya baris pertama.

### 2. Reproduksi

Munculkan bug secara konsisten di lokal atau lewat test. Kalau belum bisa:

- kumpulkan data lebih banyak (log, langkah persis, data yang dipakai, browser/perangkat);
- tambahkan log sementara di titik yang dicurigai;
- **jangan menebak-nebak perbaikan** untuk bug yang belum bisa dimunculkan.

### 3. Test yang gagal dulu

Tulis test yang **gagal karena bug ini** sebelum memperbaiki (`write-tests`). Test ini membuktikan bug ada,
dan nanti membuktikan bug sudah hilang.

Kalau test otomatis benar-benar tidak memungkinkan (mis. hanya terjadi di tampilan), tulis langkah cek manual
yang persis di laporan.

### 4. Cari akar masalah

- **Telusuri per lapisan** mengikuti alur data: UI → API → backend → database (peta di `start-here`).
  Temukan lapisan **pertama** tempat data sudah salah. Perbaikan ada di situ. Kalau ada indeks kode
  (`start-here` aturan 12), pakai untuk mengikuti jalur panggilan dari route sampai query.
- **Dulu jalan, sekarang tidak?** Cari commit penyebabnya dengan `git bisect`. [git bisect]
- **Satu dugaan sekali uji.** Ubah satu hal, lihat hasilnya, catat. Jangan mengubah banyak hal sekaligus.
- Tanyakan "kenapa" sampai ketemu penyebab yang bisa diperbaiki, bukan berhenti di gejala.

### 5. Perbaiki minimal

- Ubah sesedikit mungkin, di akar masalah.
- Bukan perbaikan: membungkam error (`try/catch` kosong), menyembunyikan tampilan yang salah, menaikkan timeout
  atau retry tanpa tahu sebabnya, mengubah test supaya lulus.

### 6. Cek dampak

- Cari pola yang sama di tempat lain, dan semua pemanggil kode yang diubah (indeks kode kalau ada, `grep`
  kalau tidak). Kalau penyebabnya sama dan masih dalam lingkup, perbaiki
  juga; kalau tidak, catat di laporan.
- **Bug sempat merusak data di produksi?** Berhenti dan laporkan ke user. Perbaikan data adalah pekerjaan
  terpisah: rencana tertulis, backup dulu (`operate-app`), dan izin user sebelum dijalankan.

### 7. Buktikan

- Test dari langkah 3 sekarang lulus, dan **seluruh** test lain tetap lulus.
- Cek nyata seperti user di langkah reproduksi.

### 8. Commit dan laporan

- Commit `fix: …` (atau konvensi proyek). Pesan menyebut **penyebab**, bukan hanya "perbaiki bug".
- Laporan bahasa sederhana: apa penyebabnya, apa yang diubah, bagaimana dicek, cara mencegahnya terulang.

## Mode rapikan (refactor)

Refactor = mengubah struktur kode **tanpa mengubah perilaku** yang terlihat dari luar. [Fowler]

1. **Kunci perilaku dulu.** Pastikan ada test yang menutup bagian yang akan dirapikan. Kalau belum ada, tulis
   *characterization test*: test yang mencatat perilaku saat ini apa adanya. [Characterization test]
2. **Langkah kecil.** Satu perubahan (pindah file, pecah fungsi, ganti nama), jalankan test, ulangi.
3. **Jangan dicampur.** Refactor dan perubahan perilaku tidak boleh di commit yang sama. Ketemu bug saat
   merapikan? Catat, selesaikan refactor, lalu perbaiki di commit `fix:` terpisah.
4. Letak baru mengikuti `structure-project`.
5. Commit `refactor: …`.

## Larangan

- Memperbaiki sebelum bisa mereproduksi.
- Menghapus, melewati (`skip`), atau melemahkan test yang gagal.
- Mengubah banyak hal sekaligus "siapa tahu salah satunya benar".
- Menjalankan skrip perbaikan data produksi tanpa backup dan izin.
- Refactor besar diselipkan di dalam perbaikan bug.

## Cek hasil

- [ ] Bug bisa direproduksi sebelum diperbaiki (atau alasan tidak bisa dicatat).
- [ ] Ada test yang dulu gagal dan sekarang lulus.
- [ ] Seluruh test lulus; hasil perintah dilampirkan di laporan.
- [ ] Penyebab tertulis di commit dan laporan.
- [ ] Mode rapikan: tidak ada test yang perlu diubah ekspektasinya.

## Sumber

- [Martin Fowler: Definition of Refactoring](https://martinfowler.com/bliki/DefinitionOfRefactoring.html).
- [Characterization test](https://en.wikipedia.org/wiki/Characterization_test) (Michael Feathers).
- [git bisect](https://git-scm.com/docs/git-bisect).
- [Google Engineering Practices: Small CLs](https://google.github.io/eng-practices/review/developer/small-cls.html): pisahkan refactor dari perubahan perilaku.
