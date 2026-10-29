---
name: change-database
description: Skill ini merancang dan mengubah database relasional dengan aman dan efisien - desain tabel, tipe data (uang, waktu, ID), constraint, index, query, transaksi, ledger saldo, migrasi tanpa downtime, masa simpan data, dan hak akses. Pakai saat menambah atau mengubah tabel/kolom/index, menulis migrasi, menulis query, menyimpan uang atau saldo, query terasa lambat, atau tabel terus membesar.
license: MIT
metadata:
  version: "0.1"
  language: id
  status: draf
---

# Mengubah database

## Tujuan

Data selalu benar karena database sendiri yang menjaga aturannya, perubahan struktur tidak pernah mematikan
aplikasi, dan query tetap cepat saat data membesar tanpa memboroskan server.

## Kapan dipakai

- Menambah/mengubah tabel, kolom, index, constraint.
- Menulis migrasi atau query.
- Menyimpan uang, saldo, atau riwayat transaksi.
- Query lambat, tabel terus membesar.

**Tidak dipakai untuk:** cache dan antrean (`build-backend`), backup dan pemulihan (`operate-app`), bentuk
JSON API (`design-api`).

## Data proyek yang dibutuhkan

- Database dan versinya, alat migrasi yang dipakai (`docs/PROJECT.md`). Belum ada alat migrasi → lihat
  `references/`, catat pilihannya sebagai ADR.

## Aturan wajib

### 1. Desain tabel

- Satu tabel untuk satu jenis hal. Nama `snake_case`, jamak: `users`, `api_keys`. [keputusan webcraft]
- Setiap tabel punya primary key dan `created_at`; tabel yang barisnya diubah juga punya `updated_at`.
- Kolom penghubung bernama `<tunggal>_id` (`user_id`) dan **selalu** diberi foreign key.
- Huruf kecil saja untuk nama tabel/kolom. [PostgreSQL: Don't Do This]

### 2. Tipe data

| Data | Pakai | Jangan |
|---|---|---|
| ID internal | `bigint` identity | `serial` [Don't Do This] |
| ID yang tampil di URL publik dan tidak boleh ditebak | UUID | angka berurutan |
| Uang | `bigint` satuan terkecil (rupiah, sen), atau `numeric` bila perlu pecahan | `float`/`real`/`double`, tipe `money` [Don't Do This] |
| Waktu | `timestamptz`, disimpan UTC | `timestamp` tanpa zona waktu [Don't Do This] |
| Teks | `text` (+ `CHECK` panjang bila perlu) | `varchar(n)`/`char(n)` sebagai bawaan [Don't Do This] |
| Pilihan tetap (status) | `text` + `CHECK (status IN (…))` | angka ajaib tanpa arti |
| Data bebas tanpa struktur | `jsonb` | `jsonb` untuk menghindari merancang kolom |

### 3. Database menjaga aturannya sendiri [PostgreSQL: Constraints]

- `NOT NULL` sebagai bawaan; boleh kosong hanya kalau memang bermakna "tidak ada".
- `UNIQUE` untuk yang tidak boleh kembar (email, kunci idempotency).
- `CHECK` untuk nilai yang harus masuk akal (`amount > 0`, `balance >= 0`).
- Foreign key dengan `ON DELETE` yang disengaja: `RESTRICT` sebagai bawaan; `CASCADE` hanya untuk data anak yang
  memang ikut terhapus bersama induknya.
- Aturan di kode saja tidak cukup: dua request bersamaan bisa lolos cek di kode, tapi tidak lolos constraint.

### 4. Index

- Index dibuat untuk query yang **benar-benar ada**: kolom di `WHERE`, `JOIN`, `ORDER BY`.
- Kolom foreign key diberi index; PostgreSQL tidak membuatnya otomatis. [PostgreSQL: Constraints]
- Index gabungan: urutan kolom mengikuti cara query memfilter.
- Index berlebihan memperlambat setiap tulis dan memakan disk. Jangan index "untuk jaga-jaga".
- Ragu? Buktikan dengan `EXPLAIN ANALYZE`. [PostgreSQL: Using EXPLAIN]

### 5. Query

- **Selalu parameter**, tidak pernah menyambung string dari input. [OWASP SQL Injection Prevention]
- Ambil kolom yang dibutuhkan saja, bukan `SELECT *`.
- **Tidak ada query di dalam perulangan** (N+1). Ambil sekaligus dengan `JOIN` atau `WHERE id = ANY($1)`.
- Daftar selalu dibatasi (`LIMIT`); data besar memakai keyset/cursor, bukan `OFFSET` besar. [Use The Index, Luke]
- `NOT EXISTS`, bukan `NOT IN (subquery)`. Rentang waktu dengan `>= awal AND < akhir`, bukan `BETWEEN`.
  [Don't Do This]

### 6. Transaksi

- Perubahan yang harus terjadi bersama (kurangi saldo + catat transaksi) ada dalam **satu transaksi**.
- Transaksi dibuat **pendek**. Tidak ada panggilan ke layanan luar (HTTP, email) di dalam transaksi.
- Angka yang diubah banyak request (saldo, stok): ubah secara atomik di database
  (`UPDATE … SET balance = balance - $1 WHERE id = $2 AND balance >= $1`) atau kunci barisnya
  (`SELECT … FOR UPDATE`). Jangan baca-hitung-tulis di kode.

### 7. Uang dan saldo

- Setiap mutasi uang dicatat sebagai **baris baru** di tabel ledger yang tidak pernah diubah atau dihapus.
- Saldo yang disimpan (kalau ada) diperbarui di transaksi yang sama dengan baris ledger, dan selalu bisa
  dihitung ulang dari ledger.
- Setiap mutasi dari luar (webhook, request user) membawa kunci unik dengan constraint `UNIQUE`, supaya tidak
  tercatat dua kali. Detail alur: `implement-payment`.

### 8. Migrasi

- Semua perubahan struktur lewat **file migrasi bernomor** di repo. Tidak ada perubahan manual di database.
- Migrasi yang sudah pernah dijalankan di mana pun **tidak diedit**; perbaikan = migrasi baru.
- Satu migrasi satu perubahan kecil.
- Perubahan yang merusak kompatibilitas (ganti nama, ganti tipe, hapus kolom) dipecah jadi tahapan
  **expand → migrate → contract** [Fowler: Parallel Change]:
  1. tambah yang baru (kode lama tetap jalan);
  2. kode menulis ke keduanya, data lama dipindahkan bertahap;
  3. kode hanya memakai yang baru;
  4. hapus yang lama di migrasi berikutnya.
- Migrasi di tabel besar tidak boleh mengunci tabel lama-lama (cara per database ada di `references/`).
- Sebelum migrasi produksi: backup (`operate-app`), uji migrasi di salinan data, **izin user**.
- Migrasi dijalankan **sebelum** kode yang membutuhkannya di-deploy.

### 9. Data yang terus bertambah

- Log, riwayat, notifikasi, sesi: tentukan **masa simpan** dan jadwal penghapusan/arsip sejak awal.
- File dan gambar disimpan di penyimpanan file, bukan di database; database hanya menyimpan alamatnya.

### 10. Akses dan koneksi

- Hanya backend yang tersambung ke database; port database tidak dibuka ke internet.
- User database aplikasi **bukan** superuser; hanya hak yang dibutuhkan.
- Aplikasi memakai **connection pool** dengan ukuran terbatas, dan setiap query punya batas waktu.
- Data pribadi disimpan seperlunya. Sandi tidak pernah disimpan apa adanya (`implement-auth`).

## Larangan

- Mengubah struktur langsung di database produksi.
- Mengedit file migrasi lama.
- `float` untuk uang; `timestamp` tanpa zona untuk waktu.
- Menghapus atau mengganti nama kolom dalam satu langkah selagi kode lama masih jalan.
- Query dengan string dari input.
- Tabel log tanpa masa simpan.

## Cek hasil

- [ ] Migrasi baru bisa dijalankan dari database kosong **dan** dari keadaan produksi sekarang.
- [ ] Constraint menolak data salah (diuji: insert yang melanggar harus gagal).
- [ ] Query baru dicek dengan `EXPLAIN` bila menyentuh tabel besar.
- [ ] Tidak ada N+1; daftar punya batas.
- [ ] Test database membersihkan datanya sendiri (`write-tests`).

## Stack

- `references/postgresql.md`: contoh SQL, migrasi tanpa mengunci tabel, ledger
- `references/go.md`: pool, transaksi, query dari Go

## Sumber

- [PostgreSQL Wiki: Don't Do This](https://wiki.postgresql.org/wiki/Don%27t_Do_This).
- [PostgreSQL: Constraints](https://www.postgresql.org/docs/current/ddl-constraints.html).
- [PostgreSQL: Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html).
- [Use The Index, Luke](https://use-the-index-luke.com/): index dan pagination tanpa offset.
- [Martin Fowler: Parallel Change](https://martinfowler.com/bliki/ParallelChange.html).
- [OWASP SQL Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html).
