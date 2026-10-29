---
name: operate-app
description: Skill ini menjaga aplikasi yang sudah berjalan - backup terenkripsi yang rutin diuji pulihkan, pemantauan uptime dan sumber daya, log dan peringatan yang berguna, penanganan insiden (termasuk kebocoran data), postmortem, runbook, serta perawatan rutin (update, rotasi rahasia, akses). Pakai saat menyiapkan atau memeriksa backup, memulihkan data, memasang pemantauan atau alert, server down atau lambat, disk penuh, sertifikat bermasalah, dugaan peretasan, atau menulis runbook.
license: MIT
metadata:
  version: "0.2"
  language: id
  status: draf
---

# Mengoperasikan aplikasi

## Tujuan

Masalah ketahuan lebih dulu oleh pemilik, bukan oleh pengguna. Data selalu bisa dipulihkan karena pemulihannya
sudah terbukti. Saat terjadi gangguan, ada langkah tertulis yang tinggal diikuti.

## Kapan dipakai

Backup dan pemulihan, pemantauan dan alert, gangguan/insiden, runbook, perawatan rutin.

**Tidak dipakai untuk:** deploy dan rollback rutin (`deploy-app`), memperbaiki bug di kode (`fix-bug`).

## Data proyek yang dibutuhkan

- Target **RPO** (berapa lama data boleh hilang) dan **RTO** (berapa lama boleh mati). Belum ada → tanyakan ke
  user dengan rekomendasi, catat di `docs/PROJECT.md`. Rekomendasi awal untuk proyek kecil/satu pengelola:

  | Jenis proyek | RPO | RTO | Artinya |
  |---|---|---|---|
  | tanpa uang (blog, profil, alat gratis) | 24 jam | 24 jam | backup harian cukup |
  | ada saldo/pembayaran | 24 jam + catatan gateway | 4 jam | backup harian; transaksi sejak backup terakhir dipulihkan dari laporan gateway pembayaran |
  | transaksi banyak per jam | ≤ 15 menit | 1-4 jam | butuh arsip WAL/PITR (ADR) |

  Angka boleh diubah user; yang penting tertulis, supaya frekuensi backup dan uji pulih punya dasar.
- Ke mana alert dikirim dan siapa yang membacanya. Proyek satu pengelola: satu kanal yang pasti dibaca di HP
  (mis. bot Telegram atau email pribadi), bukan dashboard yang harus dibuka.
- **Pemulihan data, perubahan di server produksi, dan pemberitahuan ke user selalu dengan izin user.**

## Aturan wajib

### 1. Backup [Google SRE: Data integrity]

- Yang di-backup: database **dan** file unggahan (kalau file disimpan di dalam database, backup database
  sudah mencakupnya; ukurannya diawasi). Config/rahasia disimpan terpisah di pengelola sandi milik admin, tidak
  bersama backup.
- Frekuensi mengikuti RPO. Otomatis (cron/timer), bukan manual.
- **Terenkripsi** sebelum keluar server: server hanya memegang **kunci publik**, kunci privat disimpan offline
  oleh admin.
- Disimpan di **luar server** (penyimpanan objek di penyedia lain), plus salinan lokal singkat.
- Masa simpan jelas, mis. harian 14 hari, mingguan 8 minggu, bulanan 12 bulan.
- Backup yang gagal atau tidak muncul memicu alert.
- **Uji pulihkan minimal sebulan sekali** ke database sementara, lalu cek isinya. Backup yang belum pernah
  dipulihkan belum terbukti backup. Catat tanggal dan hasil uji.
- Butuh RPO beberapa menit: arsip WAL / point-in-time recovery, dicatat sebagai ADR.

### 2. Pemantauan

Dari **luar** server (layanan uptime terpisah), setiap 1-5 menit:

- halaman utama, endpoint `/readyz` API, halaman login;
- masa berlaku sertifikat HTTPS (alert ≥ 14 hari sebelum habis).

Dari dalam server:

- disk (alert di 80%), memori, CPU, container yang restart berulang;
- backup terakhir berhasil dan tidak lebih tua dari jadwalnya.

Untuk API, empat sinyal utama [Google SRE: Monitoring]: **latensi**, **jumlah request**, **tingkat error**, dan
**kejenuhan** (antrean, koneksi pool, memori).

### 3. Log

- Log container dikumpulkan dengan rotasi/masa simpan (mis. 14-30 hari), bisa dicari berdasarkan ID request.
- Log peristiwa keamanan dan audit admin disimpan lebih lama (`build-admin-panel`).

### 4. Alert

- Setiap alert **bisa ditindaklanjuti** dan menunjuk ke runbook-nya. Alert yang selalu diabaikan dihapus atau
  diperbaiki.
- Berdasarkan gejala yang dirasakan user (situs mati, error naik, lambat), bukan setiap angka kecil.
- Dikirim ke kanal yang benar-benar dibaca pemilik (mis. Telegram/email), dengan pengelompokan supaya tidak
  banjir.

### 5. Insiden

1. **Pastikan dan ukur:** apa yang rusak, siapa yang terdampak, sejak kapan.
2. **Kabari:** pemilik/tim, dan user bila dampaknya terasa (halaman status/pengumuman).
3. **Pulihkan dulu, perbaiki kemudian:** rollback, matikan fitur yang bermasalah, tambah kapasitas. Mencari
   akar masalah dilakukan setelah layanan pulih.
4. **Simpan bukti:** log dan keadaan sebelum diubah, terutama kalau dugaan peretasan.
5. **Dugaan peretasan/kebocoran:** cabut akses penyerang, rotasi semua rahasia yang mungkin terpapar, hapus
   sesi aktif, periksa audit log. Kebocoran data pribadi wajib diberitahukan dalam 3 × 24 jam (UU PDP,
   `secure-feature`); siapkan dengan izin pemilik.
6. **Postmortem tanpa menyalahkan** [Google SRE: Postmortem culture] dalam beberapa hari: kronologi, dampak,
   akar masalah, apa yang berjalan baik/buruk, tindakan perbaikan dengan penanggung jawab dan tenggat. Disimpan
   di `docs/postmortems/`.

### 6. Runbook

Prosedur yang sering atau genting ditulis di `docs/runbooks/` dari `assets/runbook.md`, dan **sudah pernah
dicoba**:

- deploy dan rollback;
- memulihkan backup;
- rotasi rahasia (DB, kunci API, token bot, secret gateway);
- disk penuh, sertifikat gagal diperbarui, container terus restart.

### 7. Perawatan rutin

| Kapan | Apa |
|---|---|
| Mingguan | cek alert yang muncul, kapasitas disk |
| Bulanan | update OS dan image dasar, update dependensi, **uji pulihkan backup**, cek daftar akses server |
| Saat orang keluar dari tim | cabut akses SSH/panel/penyedia, rotasi rahasia yang ia ketahui |
| Tahunan atau sesuai penyedia | rotasi kunci jangka panjang |

## Larangan

- Backup tanpa enkripsi, atau kunci privat disimpan di server yang sama.
- Backup yang tidak pernah diuji pulihkan.
- Pemantauan yang hanya berjalan di server yang dipantau.
- Mencari akar masalah lama-lama sementara layanan masih mati padahal rollback tersedia.
- Memulihkan backup ke database produksi tanpa izin dan tanpa backup keadaan saat itu.

## Cek hasil

- [ ] Backup berjalan otomatis; uji pulih terakhir tercatat dan berhasil.
- [ ] Pemantau luar aktif untuk halaman utama, API, sertifikat; alert sampai ke pemilik (sudah diuji kirim).
- [ ] Runbook utama ada dan pernah dicoba.
- [ ] RPO/RTO tercatat di `docs/PROJECT.md`.

## Aset

- `assets/runbook.md`: kerangka runbook

## Stack

- `references/postgresql.md`: backup terenkripsi ke penyimpanan objek + uji pulihkan

## Sumber

- [Google SRE Book: Data Integrity](https://sre.google/sre-book/data-integrity/).
- [Google SRE Book: Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/).
- [Google SRE Book: Postmortem Culture](https://sre.google/sre-book/postmortem-culture/).
- [PostgreSQL: Backup and Restore](https://www.postgresql.org/docs/current/backup.html).
