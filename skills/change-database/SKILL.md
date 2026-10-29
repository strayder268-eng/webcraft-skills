---
name: change-database
description: Skill ini merancang dan mengubah database relasional dengan aman dan efisien - desain tabel, tipe data (uang, waktu, ID), constraint, index, query, transaksi, ledger saldo, migrasi tanpa downtime, masa simpan data, dan hak akses. Pakai saat menambah atau mengubah tabel/kolom/index, menulis migrasi, menulis query, menyimpan uang atau saldo, menyimpan token/kunci/rahasia, query terasa lambat, atau tabel terus membesar.
license: MIT
metadata:
  version: "0.4"
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

- **Selalu parameter** (`$1`, `$2`), tidak pernah menyambung string dari input. [OWASP SQL Injection Prevention]
- **Mencegah SQL injection sampai tuntas:**
  1. **Bagian yang tidak bisa jadi parameter** (nama kolom untuk `ORDER BY`, arah `ASC`/`DESC`, nama tabel):
     input user hanya dipakai sebagai **kunci** untuk memilih potongan SQL tetap dari daftar yang diizinkan.
     Nilai di luar daftar → 422. Input tidak pernah ditempel ke SQL.
  2. **Pencarian `LIKE`/`ILIKE`:** karakter `%`, `_`, dan `\` dari input di-escape, nilainya tetap lewat
     parameter, dan panjangnya dibatasi. Tanpa escape, user bisa memaksa pencarian yang memindai seluruh tabel.
  3. **Data dari database sendiri juga input.** Nilai yang pernah disimpan user lalu dibaca lagi tetap lewat
     parameter (*second-order injection*). SQL dinamis di dalam fungsi database memakai `format()` dengan `%I`
     (nama) / `%L` (nilai), atau `EXECUTE … USING`.
  4. **Dicek otomatis:** linter keamanan di CI menolak SQL yang disusun dengan format string atau
     penyambungan (cara per bahasa ada di `references/`).
  Lapisan cadangan: user database aplikasi berhak minimum (aturan 10), dan error tidak membocorkan SQL
  (`design-api` aturan 5).
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

Contoh bentuk: alat migrasi bernomor (goose untuk Go, lihat `references/`). Alat lain sah asal setiap
perubahan tercatat berurutan di repo dan bisa diulang.

**Proyek lama dengan satu file skema** (mis. `schema.sql` idempoten yang dijalankan ulang setiap deploy):
statusnya *perlu keputusan* (masuk audit sebagai temuan P2) kalau file itu berisi **operasi berbahaya** yang
ikut jalan setiap kali: `DROP`, ganti tipe, atau perubahan data (`DELETE`, `UPDATE`, `TRUNCATE`). Alasannya:
tidak ada urutan, tidak ada tahapan, dan satu baris lama yang keliru mengenai produksi di setiap deploy. Sampai
pindah alat diputuskan (ADR):

1. Perubahan baru **hanya ditambahkan di bawah**, idempoten (`IF NOT EXISTS`), dengan komentar tanggal dan
   fitur. Baris lama tidak diedit.
2. `DROP` dan ganti tipe tetap lewat expand → migrate → contract; tahap hapus baru ditulis setelah kode lama
   tidak lagi di produksi. Menghapus tabel/kolom milik fitur yang sudah dibuang hanya butuh tahap terakhir:
   kode berhenti memakainya dan sudah di-deploy, backup diambil, lalu `DROP` di perubahan berikutnya.
3. Operasi berbahaya yang **sudah ada** dibuat benar-benar sekali jalan: dibungkus penjaga yang jelas
   (mis. tabel penanda `schema_once(name)` dicek di blok `DO`), atau dipindah ke skrip satu kali terpisah
   yang tidak ikut deploy. `IF EXISTS` saja tidak cukup untuk `DELETE`/`UPDATE`, karena perintah itu tetap
   jalan setiap kali.
4. File dijalankan dua kali di database test berisi data; run kedua tidak boleh error, dan jumlah baris tabel
   utama sebelum/sesudah run kedua harus sama.

Jalur pindah ke migrasi bernomor (satu tugas sendiri, dengan backup dan izin user):

1. Ambil struktur produksi (`pg_dump --schema-only`) dan pastikan sama dengan hasil file skema di database
   kosong. Perbedaan diselesaikan dulu.
2. Jadikan hasilnya migrasi pertama (*baseline*).
3. Tandai baseline sudah berjalan di produksi **tanpa menjalankannya** (cara per alat di dokumentasi alat itu).
4. Sejak itu semua perubahan berupa file bernomor baru; file skema lama tidak diedit lagi.

### 9. Data yang terus bertambah

- Log, riwayat, notifikasi, sesi: tentukan **masa simpan** dan jadwal penghapusan/arsip sejak awal.
- File dan gambar disimpan di penyimpanan file, bukan di database; database hanya menyimpan alamatnya.

### 10. Akses dan koneksi

- Hanya backend yang tersambung ke database; port database tidak dibuka ke internet.
- User database aplikasi **bukan** superuser; hanya hak yang dibutuhkan.
- Aplikasi memakai **connection pool** dengan ukuran terbatas, dan setiap query punya batas waktu.
- Data pribadi disimpan seperlunya. Sandi tidak pernah disimpan apa adanya (`implement-auth`).

### 11. Data rahasia di database [OWASP Cryptographic Storage]

Pilih cara menyimpan berdasarkan perlu-tidaknya nilai asli dibaca lagi:

| Data | Perlu dibaca lagi? | Simpan sebagai |
|---|---|---|
| Sandi user | tidak | hash lambat: Argon2id/bcrypt (`implement-auth`) |
| Kunci API yang kita terbitkan, token sesi/reset | tidak | hash SHA-256 (nilainya acak dan panjang) |
| Kunci API milik pihak ketiga, rahasia TOTP, token OAuth, data sangat sensitif | ya | **dienkripsi** AES-256-GCM |

Aturan enkripsi:

- Pakai algoritma terautentikasi (AES-256-GCM) dari pustaka standar bahasa. Tidak membuat skema sendiri,
  tidak memakai mode ECB/CBC tanpa autentikasi.
- Nonce acak baru setiap enkripsi, dari pembangkit acak kriptografis (bukan `math/rand`/`Math.random`).
- Kunci enkripsi **tidak** di database dan tidak di repo: dari environment/secret server (atau KMS), dan
  disimpan juga oleh admin di tempat aman terpisah dari backup. Tanpa kunci itu data tidak bisa dibuka.
- Simpan **ID kunci** bersama data terenkripsi, supaya kunci bisa dirotasi: data baru memakai kunci baru,
  data lama dienkripsi ulang bertahap.
- Ikat data terenkripsi ke barisnya (*associated data*: nama tabel/kolom + ID baris), supaya nilai terenkripsi
  tidak bisa dipindah ke baris lain.

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
- [ ] Kolom rahasia tersimpan sebagai hash atau terenkripsi sesuai tabel aturan 11; tidak ada nilai asli di database.

## Stack

- `references/postgresql.md`: contoh SQL, migrasi tanpa mengunci tabel, ledger
- `references/go.md`: pool, transaksi, query dari Go

## Sumber

- [PostgreSQL Wiki: Don't Do This](https://wiki.postgresql.org/wiki/Don%27t_Do_This).
- [PostgreSQL: Constraints](https://www.postgresql.org/docs/current/ddl-constraints.html).
- [PostgreSQL: Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html).
- [Use The Index, Luke](https://use-the-index-luke.com/): index dan pagination tanpa offset.
- [Martin Fowler: Parallel Change](https://martinfowler.com/bliki/ParallelChange.html).
- [OWASP Cryptographic Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cryptographic_Storage_Cheat_Sheet.html).
- [OWASP SQL Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html).
