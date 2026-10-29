---
name: secure-feature
description: Skill ini memeriksa keamanan sebelum dan sesudah coding - threat model ringan (STRIDE), checklist OWASP Top 10:2025, header keamanan dan CSP, upload file, SSRF, pemilihan dan pembaruan library, pemindaian rahasia, serta perlindungan data pribadi (UU PDP). Pakai saat fitur menyentuh login, uang, data pribadi, file, admin, atau layanan pihak ketiga, saat menambah atau memperbarui library, saat ada dugaan celah, atau sebelum rilis.
license: MIT
metadata:
  version: "0.1"
  language: id
  status: draf
---

# Mengamankan fitur

## Tujuan

Celah ditemukan saat masih berupa rencana (murah) dan bukan setelah dieksploitasi (mahal). Setiap risiko punya
keputusan yang tercatat: dicegah, dihilangkan, dialihkan, atau diterima dengan sadar.

## Kapan dipakai

- Fitur menyentuh login, uang, data pribadi, file, admin, atau pihak ketiga (dipanggil dari `build-feature`).
- Menambah/memperbarui library.
- Ada dugaan celah, atau menjelang rilis.

**Tidak dipakai untuk:** cara membuat login/pembayaran (`implement-auth`, `implement-payment`). Skill ini
memeriksa, skill itu membangun.

## Langkah

### 1. Threat model ringan [OWASP Threat Modeling]

Jawab empat pertanyaan, hasilnya ditulis di bagian "Keamanan" spek fitur:

1. **Apa yang dibangun?** Alur data fitur: dari mana masuk, lewat mana, disimpan di mana. Tandai **batas
   kepercayaan**: browser → API, API → layanan luar, layanan luar → webhook kita, admin → sistem.
2. **Apa yang bisa salah?** Pakai STRIDE pada setiap batas:

   | Ancaman | Pertanyaan |
   |---|---|
   | **S**poofing | bisakah seseorang mengaku sebagai user/sistem lain? |
   | **T**ampering | bisakah data diubah di jalan atau di klien (nominal, ID, peran)? |
   | **R**epudiation | bisakah aksi penting disangkal karena tidak tercatat? |
   | **I**nformation disclosure | bisakah data bocor (response, log, error, URL)? |
   | **D**enial of service | bisakah fitur dibanjiri sampai mati atau mahal (email, AI, SMS)? |
   | **E**levation of privilege | bisakah user biasa melakukan aksi admin/milik orang lain? |

   Pikirkan juga penyalahgunaan aturan bisnis: nominal negatif/nol, mengulang request, dua request bersamaan
   (race condition), melompati langkah alur.
3. **Apa yang dilakukan?** Untuk tiap ancaman: cegah / hilangkan / alihkan / terima (dengan alasan). Setiap
   pencegahan punya test.
4. **Sudah cukup?** Setelah dibangun, cek ulang daftar itu terhadap kode dan test.

### 2. Checklist OWASP Top 10:2025 [OWASP Top 10]

| Risiko | Cek | Skill terkait |
|---|---|---|
| A01 Broken Access Control | izin di server, cek pemilik objek, tolak bawaan | `design-api`, `implement-auth` |
| A02 Security Misconfiguration | mode debug mati, akun/kunci bawaan diganti, header keamanan, port tertutup | `deploy-app` |
| A03 Software Supply Chain Failures | library dicek, lockfile, CI terkunci | langkah 6 |
| A04 Cryptographic Failures | HTTPS di semua jalur, hash sandi benar, tidak membuat kripto sendiri | `implement-auth`, `deploy-app` |
| A05 Injection | query berparameter, tidak menyusun perintah shell dari input, HTML di-escape | `change-database`, `build-ui` |
| A06 Insecure Design | threat model, rate limit, aturan bisnis dijaga server | langkah 1, `build-backend` |
| A07 Authentication Failures | sesi, sandi, 2FA, batas percobaan | `implement-auth` |
| A08 Software or Data Integrity Failures | tanda tangan webhook diverifikasi, image/artefak dari CI sendiri | `implement-payment`, `deploy-app` |
| A09 Security Logging and Alerting Failures | peristiwa penting dicatat, ada peringatan | `build-backend`, `operate-app` |
| A10 Mishandling of Exceptional Conditions | gagal tertutup, error tidak bocor, timeout | `build-backend` |

### 3. Header keamanan [OWASP HTTP Headers]

| Header | Nilai |
|---|---|
| `Strict-Transport-Security` | `max-age=63072000; includeSubDomains` (tambah `preload` hanya setelah yakin semua subdomain HTTPS) |
| `Content-Security-Policy` | kebijakan ketat berbasis nonce; tidak ada skrip inline tanpa nonce |
| `X-Content-Type-Options` | `nosniff` |
| `frame-ancestors` di CSP (atau `X-Frame-Options: DENY`) | `'none'` kecuali memang perlu di-embed |
| `Referrer-Policy` | `strict-origin-when-cross-origin` |
| `Permissions-Policy` | matikan fitur yang tidak dipakai: `camera=(), microphone=(), geolocation=()` |
| `Server`, `X-Powered-By` | dihapus |

`X-XSS-Protection` tidak dipasang (atau `0`). CSP baru dipasang dulu dalam mode `Report-Only`, diamati, lalu
ditegakkan.

### 4. Upload file [OWASP File Upload]

- Daftar jenis yang diizinkan (ekstensi **dan** isi file dicek), batas ukuran.
- Nama file diganti nama acak; tidak memakai nama/path dari user.
- Disimpan di penyimpanan file terpisah, bukan di folder yang dijalankan server.
- Disajikan dengan `Content-Disposition: attachment` atau dari domain terpisah; SVG/HTML dari user tidak
  disajikan inline.
- Metadata gambar (lokasi GPS) dibuang bila gambar ditampilkan ke orang lain.

### 5. Server mengambil URL dari user (SSRF) [OWASP SSRF]

Kalau server mengunduh/memanggil URL yang diberikan user (webhook user, pratinjau tautan, impor gambar):
domain diizinkan lewat daftar, alamat IP internal/privat ditolak **setelah** DNS di-resolve dan di setiap
redirect, batas waktu dan ukuran.

### 6. Library [OWASP Vulnerable Dependency Management]

Sebelum menambah:

- [ ] Benar-benar perlu? Bisa dengan fitur bawaan bahasa/framework?
- [ ] Nama paket persis (waspada nama mirip/typosquatting).
- [ ] Masih dirawat (rilis terakhir, jumlah pengelola, isu keamanan yang terbuka).
- [ ] Lisensi cocok dengan proyek.
- [ ] Ukuran wajar (frontend: pengaruh ke ukuran JavaScript).
- [ ] Tidak ada kerentanan yang diketahui.

Selalu: lockfile di-commit, instal persis sesuai lockfile di CI, pemindai kerentanan di CI, pembaruan rutin,
kerentanan kritis ditambal secepatnya.

### 7. Rahasia

- Pemindai rahasia di CI (dan sebaiknya sebelum commit).
- Rahasia yang pernah ter-commit atau terkirim di chat **dianggap bocor**: ganti (rotasi) di penyedianya.
  Menghapusnya dari git tidak cukup.
- Rahasia berbeda untuk setiap lingkungan.

### 8. Data pribadi [UU PDP No. 27 Tahun 2022]

- Kumpulkan **seperlunya**, dengan tujuan yang ditulis di kebijakan privasi.
- Persetujuan terpisah untuk pemasaran.
- User bisa melihat, mengunduh, memperbaiki, dan menghapus datanya (hapus akun).
- Masa simpan ditentukan; data yang tidak diperlukan dihapus.
- Kebocoran data pribadi: UU PDP mewajibkan pemberitahuan tertulis kepada pemilik data dan lembaga paling
  lambat 3 × 24 jam (Pasal 46). Siapkan prosedurnya di `operate-app`. Ini ringkasan, bukan nasihat hukum.

## Larangan

- Menganggap fitur aman karena "tidak ada yang tahu URL-nya".
- Keamanan yang hanya dijaga frontend.
- Menambah library tanpa checklist langkah 6.
- Menunda rotasi rahasia yang bocor.

## Cek hasil

- [ ] Bagian "Keamanan" di spek berisi alur, batas kepercayaan, ancaman, dan keputusan.
- [ ] Setiap pencegahan punya test.
- [ ] Pemindai kerentanan dan pemindai rahasia lulus di CI.
- [ ] Header keamanan terpasang (cek dengan `curl -I`).

## Stack

- `references/nextjs.md`: CSP dengan nonce, header di Next.js
- `references/go.md`: middleware header, `govulncheck`

## Sumber

- [OWASP Threat Modeling Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Threat_Modeling_Cheat_Sheet.html).
- [OWASP Top 10:2025](https://top10.owasp.org/2025).
- [OWASP HTTP Headers Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/HTTP_Headers_Cheat_Sheet.html).
- [OWASP Content Security Policy Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Content_Security_Policy_Cheat_Sheet.html).
- [OWASP File Upload Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html).
- [OWASP SSRF Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html).
- [OWASP Vulnerable Dependency Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Vulnerable_Dependency_Management_Cheat_Sheet.html).
- [gitleaks](https://github.com/gitleaks/gitleaks): pemindai rahasia.
- [UU No. 27 Tahun 2022 tentang Pelindungan Data Pribadi](https://peraturan.bpk.go.id/Details/229798/uu-no-27-tahun-2022).
