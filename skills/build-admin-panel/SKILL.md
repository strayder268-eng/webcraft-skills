---
name: build-admin-panel
description: Skill ini membangun panel admin yang aman dan bisa diaudit - area terpisah, peran dengan hak minimum, 2FA wajib, konfirmasi ulang untuk aksi sensitif, audit log yang tidak bisa diubah, aksi lewat aturan bisnis yang sama dengan user, persetujuan dua orang untuk aksi berisiko tinggi, dan perlindungan data pribadi. Pakai saat membuat atau mengubah halaman/endpoint admin, menambah peran admin, aksi admin atas uang/akun/pengaturan, atau melihat dan mengekspor data user.
license: MIT
metadata:
  version: "0.1"
  language: id
  status: draf
---

# Panel admin

## Tujuan

Admin bisa mengelola sistem tanpa bisa (sengaja atau tidak) melewati aturan bisnis. Setiap tindakan admin
tercatat dan bisa ditelusuri, dan akun admin yang dibobol tidak langsung berarti seluruh sistem jebol.

## Kapan dipakai

Halaman/endpoint admin, peran admin, aksi admin atas uang/akun/pengaturan, melihat/mengekspor data user.

**Tidak dipakai untuk:** dasar login, sesi, 2FA (`implement-auth`, yang dipakai di sini), tampilan komponen
(`build-ui`).

## Data proyek yang dibutuhkan

- Daftar peran admin dan hak masing-masing (`docs/PROJECT.md`). Belum ada → usulkan tabel peran (aturan 2) dan
  minta persetujuan user.

## Aturan wajib

### 1. Area terpisah

- Halaman admin di area sendiri (route `/admin` atau subdomain), endpoint admin di awalan sendiri
  (`/api/v1/admin/*`) yang dijaga satu middleware peran.
- Sesi admin lebih ketat: cookie `SameSite=Strict`, batas total ≤ 24 jam dan tidak aktif ≤ 1 jam (NIST AAL2,
  `implement-auth`).
- Halaman admin `noindex`.
- Disarankan lapisan tambahan di depan admin: akses lewat proxy zero-trust/VPN atau daftar IP. Kalau ada,
  catat di ADR.
- Tampilan admin punya penanda jelas (warna aksen/label "ADMIN") supaya admin sadar sedang di area berisiko.

### 2. Peran dengan hak minimum [OWASP Authorization]

Contoh pembagian (sesuaikan proyek):

| Peran | Boleh |
|---|---|
| `support` | melihat user dan riwayat (data pribadi disamarkan), tidak bisa mengubah uang |
| `finance` | memeriksa dan menyetujui top-up manual, refund, koreksi saldo |
| `operator` | pengaturan teknis (model, harga, layanan) |
| `superadmin` | semua di atas + mengubah peran admin |

- **Tolak secara bawaan.** Setiap endpoint admin menyatakan peran yang diizinkan; dicek di server di setiap
  request.
- Jumlah `superadmin` sesedikit mungkin, tapi minimal dua orang supaya tidak terkunci.
- Admin tidak bisa menaikkan perannya sendiri.

### 3. 2FA dan konfirmasi ulang

- **2FA wajib** untuk setiap akun dengan peran admin. Belum aktif → dipaksa mendaftar sebelum bisa masuk panel.
- Aksi sensitif meminta kode 2FA lagi (berlaku beberapa menit): ubah peran, ubah saldo, setujui/tolak
  pembayaran, refund, ubah harga/pengaturan uang/keamanan, ekspor data, hapus data.

### 4. Aksi lewat jalur yang sama

- Aksi admin memanggil **service yang sama** dengan aplikasi (validasi, ledger, status satu arah). Tidak ada
  endpoint "bypass" dan tidak ada edit langsung ke database produksi lewat konsol.
- Ubah saldo = baris ledger `adjustment` dengan alasan wajib (`implement-payment`), bukan menimpa angka saldo.

### 5. Audit log [OWASP Logging]

Setiap aksi tulis admin mencatat:

| Field | Isi |
|---|---|
| waktu | `timestamptz` |
| pelaku | ID admin + perannya saat itu |
| aksi | nama tetap, mis. `user.role.update`, `balance.adjust`, `topup.approve` |
| sasaran | jenis + ID objek |
| sebelum / sesudah | perubahan field (data rahasia disamarkan) |
| alasan | wajib untuk aksi uang dan peran |
| asal | IP, user agent, ID request |

- Tabel audit **hanya bisa ditambah**: user database aplikasi tidak punya hak `UPDATE`/`DELETE`
  (`change-database`).
- Disimpan lama (mis. ≥ 1 tahun), terlihat oleh `superadmin`, tidak bisa disunting dari panel.
- Ditulis di transaksi yang sama dengan aksinya: aksi berhasil tanpa audit tidak boleh terjadi.

### 6. Aksi berbahaya

- Dialog konfirmasi yang menyebut **akibat persisnya** ("Saldo user A berubah dari Rp 50.000 menjadi
  Rp 0").
- Hapus permanen/aksi massal: ketik ulang nama/jumlah untuk konfirmasi, tampilkan pratinjau, batasi jumlah.
- Lebih baik nonaktifkan/arsipkan daripada hapus permanen.

### 7. Persetujuan dua orang (opsional, ADR)

Untuk aksi berisiko sangat tinggi (koreksi saldo di atas batas tertentu, memberi peran `superadmin`): admin
pertama mengajukan, admin **lain** menyetujui. Keduanya tercatat di audit log.

### 8. Data pribadi

- Tampilkan seperlunya; data sensitif disamarkan secara bawaan (`a***@gmail.com`).
- Membuka data lengkap butuh alasan dan tercatat di audit log.
- Ekspor data dibatasi peran, tercatat, dan tidak berisi rahasia (hash sandi, token).

### 9. Masuk sebagai user (impersonation)

Hindari. Kalau benar-benar perlu: hanya baca, waktu terbatas, ada banner jelas selama mode itu, tercatat di
audit log, dan tidak bisa melakukan aksi uang.

### 10. Pemantauan

Peringatan ke `superadmin` (`operate-app`) untuk pola aneh: banyak perubahan peran, koreksi saldo besar, login
admin dari tempat baru, gagal 2FA berulang.

## Larangan

- Endpoint admin tanpa cek peran di server (hanya menyembunyikan menu).
- Akun admin tanpa 2FA.
- Aksi admin yang menulis langsung ke tabel tanpa lewat service.
- Audit log yang bisa diubah/dihapus, atau aksi yang lolos tanpa tercatat.
- Satu akun admin bersama untuk banyak orang.

## Cek hasil

- [ ] Test: setiap endpoint admin menolak user biasa (403) dan menolak peran yang tidak berhak.
- [ ] Test: aksi sensitif tanpa konfirmasi ulang ditolak.
- [ ] Test: setiap aksi tulis admin menghasilkan satu baris audit; user DB aplikasi tidak bisa `UPDATE`/`DELETE`
      tabel audit.
- [ ] Admin tanpa 2FA tidak bisa membuka panel.

## Sumber

- [OWASP Authorization Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html): hak minimum, tolak bawaan.
- [OWASP Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html): isi dan perlindungan log audit.
- [NIST SP 800-63B-4](https://pages.nist.gov/800-63-4/sp800-63b.html): batas sesi dan reautentikasi AAL2.
- [OWASP ASVS](https://owasp.org/www-project-application-security-verification-standard/): verifikasi kontrol akses dan audit.
