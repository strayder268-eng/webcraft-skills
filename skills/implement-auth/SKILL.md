---
name: implement-auth
description: Skill ini membuat fitur akun yang aman - daftar, login (email+sandi, Google, Telegram, atau penyedia OpenID Connect lain), sesi dan cookie, logout, reset sandi, verifikasi email, 2FA/TOTP, konfirmasi ulang untuk aksi sensitif, peran dan izin, kunci API, JWT, serta pencegahan CSRF dan penebakan akun. Pakai saat menambah atau mengubah login, pendaftaran, sesi, lupa sandi, 2FA, peran user/admin, halaman yang wajib login, atau menghubungkan akun ke Google/Telegram, menerbitkan kunci API, atau memakai JWT.
license: MIT
metadata:
  version: "0.2"
  language: id
  status: draf
---

# Login, sesi, dan izin

## Tujuan

Hanya pemilik akun yang bisa masuk, sesi tidak bisa dicuri atau dipalsukan, dan setiap request hanya bisa
melakukan apa yang memang diizinkan untuk pemanggilnya.

## Kapan dipakai

Daftar, login, logout, sesi, reset sandi, verifikasi email, 2FA, peran/izin, login lewat penyedia luar.

**Tidak dipakai untuk:** panel admin secara keseluruhan (`build-admin-panel`, yang memakai aturan di sini),
pengiriman email/OTP (`send-notifications`).

## Data proyek yang dibutuhkan

- Metode login yang dipilih (ADR). Belum ada → rekomendasikan lalu catat (langkah 1).
- Peran yang ada di proyek (`docs/PROJECT.md`).

## Aturan wajib

### 1. Memilih metode login (ADR)

- Login lewat penyedia (Google, Telegram, dll. dengan OpenID Connect) mengurangi risiko menyimpan sandi.
  Email + sandi hanya kalau memang dibutuhkan.
- Satu user bisa punya beberapa cara login. Simpan di tabel identitas terpisah:
  `(user_id, provider, subject)` dengan `UNIQUE (provider, subject)`.
- Identitas dari penyedia dikenali dari `sub`, **bukan** dari email.
- Menyambungkan cara login baru ke akun yang sudah ada hanya saat user **sudah login**. Tidak pernah otomatis
  karena email-nya sama, karena itu bisa dipakai mengambil alih akun.

### 2. Sandi (kalau dipakai) [NIST SP 800-63B-4, OWASP Password Storage]

- Panjang minimal **15 karakter** kalau sandi satu-satunya faktor; minimal **8** kalau akun memakai 2FA.
  Panjang maksimal yang diterima minimal 64.
- **Tidak ada** aturan komposisi (wajib huruf besar/angka/simbol) dan **tidak ada** kewajiban ganti berkala.
- Sandi baru dicek terhadap daftar sandi umum/bocor.
- Izinkan tempel (paste) dan pengelola sandi.
- Disimpan sebagai hash **Argon2id** (m=19 MiB, t=2, p=1 sebagai minimum) atau bcrypt (cost ≥ 10, ingat batas
  72 byte). Tidak pernah dienkripsi bolak-balik, tidak pernah dicatat di log.

### 3. Tidak membocorkan keberadaan akun

- Login gagal: satu pesan untuk semua sebab ("Email atau sandi salah").
- Daftar dan lupa sandi: "Kalau email ini terdaftar, kami sudah mengirim tautan." Waktu respons dibuat mirip.

### 4. Batas percobaan

- Rate limit login per IP **dan** per akun yang dicoba (`build-backend`).
- Gagal beruntun → jeda makin lama, atau tantangan anti-bot. Batas mutlak NIST: maksimal 100 kegagalan beruntun
  sebelum akun dikunci dan perlu pemulihan.

### 5. Login lewat penyedia (OAuth 2.0 / OpenID Connect) [RFC 9700]

- Alur **Authorization Code + PKCE (S256)**, dengan `state` (cegah CSRF) dan `nonce` (cegah replay).
  Tidak memakai implicit flow.
- `redirect_uri` didaftarkan persis di penyedia dan dicocokkan persis.
- Penukaran `code` → token dilakukan di **server**. Client secret hanya di server.
- `id_token` diverifikasi: tanda tangan (kunci dari JWKS penyedia), `iss`, `aud` = client ID kita, `exp`, dan
  `nonce`.
- Email dari penyedia hanya dipakai kalau penyedia menyatakannya terverifikasi (`email_verified`).
- Detail per penyedia: `references/google.md`, `references/telegram.md`.

### 6. Sesi [OWASP Session Management]

- Untuk web, pakai **sesi di server**: ID acak ≥128 bit, disimpan dalam bentuk hash di database/Redis,
  dikirim lewat cookie. Token tidak disimpan di `localStorage`.
- Cookie: `HttpOnly`, `Secure`, `SameSite=Lax` (`Strict` untuk area admin), `Path=/`, nama berawalan
  `__Host-`, tanpa atribut `Domain`.
- Kalau login memang harus berlaku di beberapa subdomain (mis. `app.` dan `api.`), cookie butuh `Domain` dan
  tidak bisa memakai `__Host-`: pakai awalan `__Secure-`, catat di ADR, dan perlakukan **semua** subdomain
  domain itu sebagai tepercaya (subdomain mana pun bisa membaca/menimpa cookie itu). Area admin tetap
  `__Host-` di host-nya sendiri.
- ID sesi **diganti** saat login dan saat peran/izin berubah.
- Ada batas waktu tidak aktif dan batas waktu total. Area uang/admin mengikuti NIST AAL2: total ≤ 24 jam,
  tidak aktif ≤ 1 jam. Area biasa boleh lebih panjang ("ingat saya"), dicatat di ADR.
- Logout menghapus sesi **di server**, bukan hanya cookie. Sediakan "keluar dari semua perangkat".

### 7. CSRF [OWASP CSRF Prevention]

- `GET` tidak pernah mengubah data (`design-api`).
- Request yang mengubah data dengan cookie: cookie `SameSite` **plus** cek header `Origin` cocok dengan daftar
  origin kita (atau token CSRF).

### 8. Reset sandi dan verifikasi email [OWASP Forgot Password]

- Token acak, sekali pakai, disimpan sebagai hash, berlaku singkat (mis. 30-60 menit).
- Tautan memakai domain dari config, **bukan** dari header `Host` request.
- Setelah sandi direset: semua sesi lama dihapus, user diberi tahu lewat email.

### 9. 2FA [RFC 6238, OWASP MFA]

- TOTP (aplikasi authenticator) atau passkey. **Wajib untuk admin.**
- Saat mengaktifkan: berikan **kode cadangan** sekali tampil, disimpan sebagai hash, masing-masing sekali pakai.
- Rahasia TOTP disimpan terenkripsi. Kode yang sudah dipakai tidak bisa dipakai lagi di jendela waktu yang sama.
  Percobaan kode diberi rate limit.

### 10. Konfirmasi ulang (step-up)

Aksi sensitif meminta sandi/kode 2FA lagi meski sesi masih aktif: ganti email, ganti sandi, matikan 2FA,
tarik/kirim uang, aksi admin berisiko.

### 11. Izin [OWASP Authorization]

- **Tolak secara bawaan**; izin dicek di **server** di setiap request.
- Peran disimpan di database, tidak di data yang bisa diubah klien. Peran sedikit dan jelas
  (mis. `user`, `admin`).
- Setiap akses objek berdasarkan ID mengecek pemiliknya (`design-api` aturan 7).
- Pengecekan di frontend (menyembunyikan menu) hanya untuk tampilan, bukan keamanan.

### 12. Kunci API untuk user/mesin

Kalau proyek menerbitkan kunci API (untuk dipakai program, bukan browser):

- Dibuat dari pembangkit acak kriptografis, minimal 128 bit, dengan **awalan** yang mudah dikenali
  (`sk-live-…`, `sk-test-…`). Awalan membantu pemindai rahasia mengenali kunci yang bocor.
- **Ditampilkan sekali** saat dibuat. Yang disimpan hanya hash-nya (SHA-256 cukup karena kuncinya acak dan
  panjang), plus potongan untuk ditampilkan (`sk-live-…a1b2`).
- Dikirim di header `Authorization: Bearer …`, **tidak pernah** di URL/query (URL tercatat di log dan riwayat).
- Setiap kunci punya pemilik, nama, cakupan izin (scope) seperlunya, waktu dipakai terakhir, dan bisa dicabut
  kapan saja. Masa berlaku opsional, dianjurkan untuk kunci berizin besar.
- Rate limit dan batas biaya **per kunci** (`build-backend`, `secure-feature`).
- Log hanya mencatat potongan kunci.
- Kunci tidak berlaku untuk endpoint browser/admin, dan cookie sesi tidak berlaku untuk endpoint kunci API
  (dua jalur autentikasi dipisah).

### 13. JWT dan token mandiri [RFC 8725]

Untuk web, sesi di server lebih disarankan (aturan 6). Kalau JWT tetap dipakai (mis. antarlayanan):

- Algoritma ditentukan server dari daftar tetap; token dengan `alg: none` atau algoritma lain ditolak.
  Kunci simetris (HS256) dan asimetris (RS/ES) tidak boleh tertukar.
- Selalu cek `iss`, `aud`, `exp` (dan `nbf` kalau ada). Masa berlaku singkat (menit, bukan hari).
- Isi JWT bisa dibaca siapa pun: **tidak** berisi data rahasia/pribadi.
- Ada cara mencabut (daftar token dicabut atau masa berlaku singkat + refresh token yang disimpan di server).
- Kunci tanda tangan dirotasi lewat `kid`/JWKS.
- Pakai pustaka JWT yang teruji, tidak mem-parse/memverifikasi sendiri.

### 14. Catatan peristiwa

Dicatat (`build-backend`): login berhasil/gagal, logout, ganti sandi/email, aktif/nonaktif 2FA, reset sandi,
perubahan peran. Ganti sandi/email/2FA juga diberitahukan ke user.

## Larangan

- Mengenali user dari email penyedia alih-alih `sub`.
- Menggabungkan akun otomatis karena email sama.
- Token sesi di `localStorage` atau URL.
- Pesan "email tidak terdaftar" / "sandi salah" yang dibedakan.
- Membuat sistem kriptografi sendiri; pakai pustaka yang teruji.
- Login Telegram model lama (hash widget) untuk proyek baru.

## Cek hasil

- [ ] Test: login benar/salah, sesi kedaluwarsa, logout menghapus sesi server, akses objek milik orang lain
      ditolak, request tanpa `Origin` yang benar ditolak.
- [ ] Test OIDC: `state`/`nonce` salah ditolak, `aud`/`iss` salah ditolak, token kedaluwarsa ditolak.
- [ ] Cookie sesi punya atribut yang benar (cek di browser).
- [ ] Test kunci API: kunci dicabut ditolak, kunci di query string ditolak, hanya hash yang tersimpan.
- [ ] Tidak ada sandi/token di log.

## Stack

- `references/go.md`: hash sandi, cookie, cek Origin, OIDC dengan PKCE
- `references/google.md`: Google Sign-In (OIDC)
- `references/telegram.md`: Log in with Telegram (OIDC)
- `references/nextjs.md`: halaman login dan proteksi tampilan

## Sumber

- [NIST SP 800-63B-4](https://pages.nist.gov/800-63-4/sp800-63b.html): sandi, batas percobaan, reautentikasi.
- [RFC 9700: OAuth 2.0 Security Best Current Practice](https://www.rfc-editor.org/rfc/rfc9700.html).
- [RFC 8725: JSON Web Token Best Current Practices](https://www.rfc-editor.org/rfc/rfc8725.html).
- [RFC 6238: TOTP](https://www.rfc-editor.org/rfc/rfc6238.html).
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html).
- [OWASP Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html).
- [OWASP Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html).
- [OWASP CSRF Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html).
- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html).
- [OWASP Multifactor Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Multifactor_Authentication_Cheat_Sheet.html).
- [OWASP Authorization Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html).
