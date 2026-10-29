---
name: implement-payment
description: Skill ini membuat alur pembayaran dan saldo yang tidak bisa dicurangi - harga dihitung di server, invoice ke payment gateway, notifikasi/webhook bertanda tangan, cek status ke gateway, idempotency, status pembayaran satu arah, ledger saldo, transfer manual yang diverifikasi admin, refund, dan rekonsiliasi. Pakai saat menambah atau mengubah top-up, checkout, langganan, saldo/kredit, webhook pembayaran, refund, atau integrasi payment gateway (DOKU, Midtrans, Xendit, Stripe, dll.).
license: MIT
metadata:
  version: "0.1"
  language: id
  status: draf
---

# Pembayaran dan saldo

## Tujuan

Uang hanya tercatat masuk kalau gateway benar-benar mengonfirmasi, tidak pernah tercatat dua kali, jumlahnya
selalu sesuai pesanan, dan setiap rupiah bisa ditelusuri.

## Kapan dipakai

Top-up, checkout, langganan, saldo/kredit, webhook pembayaran, refund, transfer manual.

**Tidak dipakai untuk:** skema tabel secara umum (`change-database`, termasuk contoh ledger), panel admin
secara umum (`build-admin-panel`).

## Data proyek yang dibutuhkan

- Gateway yang dipakai dan dokumentasinya (ADR). Gateway tanpa file `references/` → baca dokumentasi resminya
  dan beri tahu user bahwa belum ada panduan teruji (`CATALOG.md`).
- Mata uang dan aturan bisnis (minimal/maksimal top-up, biaya).

## Aturan wajib

### 1. Data kartu tidak pernah menyentuh server kita

Pakai halaman bayar milik gateway (hosted checkout/redirect) atau komponen resmi mereka. Server tidak menerima,
menyimpan, atau mencatat nomor kartu. Ini menjaga kewajiban PCI DSS tetap minimal. [PCI DSS]

### 2. Harga dihitung di server [OWASP Payment Gateway]

Klien hanya mengirim **apa** yang dibeli (ID paket, jumlah). Nominal, diskon, biaya, dan mata uang dihitung di
server dari data di database. Nominal dari klien tidak pernah dipakai.

### 3. Status pembayaran satu arah

```
pending ──► paid
   │
   ├──► failed
   └──► expired
paid ──► refunded (sebagian / penuh)
```

- Perpindahan status dilakukan atomik dengan syarat status asal
  (`UPDATE … SET status='paid' WHERE id=$1 AND status='pending'`). Kalau 0 baris berubah, berarti sudah
  diproses; jangan kreditkan lagi.
- Tidak pernah mundur (`paid` → `pending`). Notifikasi yang datang terlambat untuk pembayaran yang sudah
  `expired` di sisi kita tetap dicek ke gateway, bukan dibuang. [OWASP Payment Gateway]

### 4. Membuat pembayaran

1. Simpan baris pembayaran `pending` dengan ID unik milik kita, nominal, mata uang, user.
2. Panggil API gateway dari server dengan ID itu sebagai nomor invoice/referensi dan kunci idempotency
   (`build-backend`: timeout, retry hanya kalau idempoten).
3. Kembalikan URL bayar ke klien.

### 5. Halaman kembali (return/redirect) tidak mengkreditkan

Parameter di URL kembali bisa dipalsukan. Halaman itu hanya **menampilkan status** dari database kita (dan
boleh memicu cek status di server). Kredit hanya terjadi lewat aturan 6 atau 7.

### 6. Notifikasi dari gateway (webhook)

- **Verifikasi tanda tangan** sesuai dokumentasi gateway, terhadap **body mentah** (sebelum di-parse), dengan
  perbandingan waktu-konstan. Gagal → 401, tidak diproses.
- Cek **jumlah, mata uang, nomor invoice, dan ID merchant** cocok dengan baris pembayaran kita.
- **Idempoten:** simpan ID notifikasi/event dengan constraint `UNIQUE`. Notifikasi yang sama datang lagi →
  jawab sukses tanpa memproses ulang. Gateway memang bisa mengirim ulang. [Stripe: Webhooks]
- Jawab cepat (2xx) setelah tercatat; pekerjaan lanjutan (email struk) lewat antrean (`build-backend`).
- Endpoint webhook dikecualikan dari cek sesi/CSRF, tapi tetap dijaga tanda tangan dan rate limit.

### 7. Cek status ke gateway

Selain menunggu webhook, server bisa menanyakan status ke API gateway (saat user kembali ke halaman, dan secara
berkala untuk yang lama `pending`). Hasilnya diperlakukan seperti notifikasi: cocokkan jumlah/invoice, lalu
perpindahan status atomik. [OWASP Payment Gateway]

### 8. Mengkreditkan saldo

Dalam **satu transaksi database**: ubah status `pending → paid` + tulis baris ledger (kunci idempotency =
ID pembayaran) + perbarui saldo. Pola tabel: `change-database` (ledger). Saldo tidak pernah diubah tanpa baris
ledger.

### 9. Memakai saldo

Kurangi saldo secara atomik dengan syarat cukup (`… WHERE balance >= $1`), tulis ledger negatif di transaksi
yang sama. Saldo tidak boleh negatif (constraint `CHECK`).

### 10. Transfer manual / QRIS statis

- User mengunggah bukti → status `pending_review`.
- Admin memeriksa mutasi rekening, lalu menyetujui/menolak lewat panel (`build-admin-panel`) dengan
  konfirmasi ulang 2FA. Persetujuan memakai jalur kredit yang sama (aturan 8) dan tercatat di audit log.
- Nominal unik (mis. tambahan kode 3 digit) membantu mencocokkan mutasi.

### 11. Refund dan koreksi

- Refund lewat API gateway, dicatat sebagai baris ledger negatif yang menunjuk pembayaran asal.
- Koreksi saldo manual oleh admin: baris ledger jenis `adjustment` dengan alasan wajib dan audit log. Tidak ada
  edit langsung ke angka saldo.

### 12. Rekonsiliasi

- Harian: bandingkan pembayaran `paid` di database dengan laporan transaksi gateway.
- Berkala: jumlah ledger per akun = saldo tersimpan.
- Selisih dilaporkan ke admin, tidak diperbaiki otomatis.

### 13. Rahasia dan log

- Kunci API/secret gateway hanya di environment server; berbeda untuk sandbox dan produksi.
- Log mencatat ID pembayaran, invoice, status, hasil verifikasi; **tidak** mencatat kunci, tanda tangan
  mentah, atau data kartu.

## Larangan

- Mengkreditkan dari halaman kembali atau dari data klien.
- Memproses webhook tanpa verifikasi tanda tangan, atau setelah body diubah/di-parse ulang.
- Nominal dari klien.
- `UPDATE saldo` tanpa baris ledger.
- `float` untuk uang.
- Memakai kunci produksi di lingkungan pengembangan.

## Cek hasil

- [ ] Test: webhook tanda tangan salah ditolak; webhook dikirim dua kali hanya mengkreditkan sekali; nominal
      tidak cocok ditolak; dua request bayar bersamaan dengan saldo pas hanya satu yang berhasil.
- [ ] Alur penuh dicoba di **sandbox** gateway: bayar sukses, gagal, kedaluwarsa.
- [ ] Ledger dan saldo cocok setelah semua test.
- [ ] `secure-feature` sudah dijalankan untuk fitur ini.

## Stack

- `references/go.md`: verifikasi tanda tangan, webhook idempoten, kredit atomik

Panduan per gateway belum tersedia (lihat `CATALOG.md`).

## Sumber

- [OWASP Third Party Payment Gateway Integration Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Third_Party_Payment_Gateway_Integration_Cheat_Sheet.html).
- [Stripe: Receive events in your webhook endpoint](https://docs.stripe.com/webhooks): tanda tangan, body mentah, event ganda.
- [Stripe: Idempotent requests](https://docs.stripe.com/api/idempotent_requests).
- [PCI Security Standards Council](https://www.pcisecuritystandards.org/).
