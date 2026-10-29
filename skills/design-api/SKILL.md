---
name: design-api
description: Skill ini merancang dan menjaga kontrak API HTTP/JSON antara frontend dan backend - URL, method, status code, format error seragam (RFC 9457), validasi input, izin per endpoint, bentuk data uang/waktu/ID, pagination, idempotency, rate limit, kompatibilitas versi, serta SSE/WebSocket. Pakai saat menambah atau mengubah endpoint, mengubah bentuk request/response, memanggil API dari frontend, atau menyeragamkan pesan error.
license: MIT
metadata:
  version: "0.3"
  language: id
  status: draf
---

# Merancang API

## Tujuan

Semua endpoint berperilaku dengan cara yang sama: mudah ditebak, aman secara bawaan, dan bisa diubah tanpa
merusak pemakai lama.

## Kapan dipakai

- Menambah atau mengubah endpoint.
- Mengubah bentuk request/response.
- Membuat pemanggil API di frontend.

**Tidak dipakai untuk:** logika di balik endpoint (`build-backend`), skema tabel (`change-database`), detail
login/sesi (`implement-auth`), webhook pembayaran (`implement-payment`).

## Data proyek yang dibutuhkan

- Konvensi API yang sudah tercatat di `docs/PROJECT.md` atau ADR (prefix, versi, gaya nama field). Kalau
  belum ada, pakai bawaan skill ini dan catat sebagai ADR.

## Aturan wajib

### 1. Kontrak dulu

Sebelum ditulis kodenya, setiap endpoint punya kontrak di spek fitur: method, path, siapa yang boleh memanggil,
request, response sukses, kemungkinan error.

### 2. URL

- Kata benda jamak, huruf kecil, kebab-case: `/api/v1/api-keys/{id}`.
- Aksi yang bukan buat/baca/ubah/hapus: `POST /api/v1/orders/{id}/cancel`.
- API untuk pihak luar (dipanggil program/klien di luar proyek) diberi versi di path (`/v1`). Konsisten di
  semua endpoint. API internal yang hanya dipanggil frontend proyek sendiri boleh tanpa versi, karena keduanya
  dirilis bersama.

**Kontrak pihak lain.** Kalau API sengaja meniru kontrak layanan lain supaya SDK-nya bisa dipakai (mis.
kompatibel OpenAI, S3), bentuk URL, error, dan data **mengikuti kontrak itu**, bukan aturan 2, 5, 8, dan 9 di
skill ini, **termasuk status code** (aturan 4) bila kontrak itu menentukannya. Catat kontrak mana yang diikuti
di ADR atau dokumen keputusan yang sudah ada. Field response tambahan tetap aman (aturan 12) selama SDK pihak
itu mengabaikan field yang tidak dikenalnya. Aturan keamanan (6, 7, 10, 11) tetap berlaku.

### 3. Method [RFC 9110]

| Method | Untuk | Sifat |
|---|---|---|
| `GET` | membaca | **aman**: tidak boleh mengubah data apa pun |
| `POST` | membuat / aksi | tidak idempoten (lihat aturan 10) |
| `PUT` | mengganti utuh | idempoten |
| `PATCH` | mengubah sebagian | |
| `DELETE` | menghapus | idempoten |

### 4. Status code

| Kode | Kapan |
|---|---|
| 200 | berhasil, ada isi |
| 201 | berhasil membuat; sertakan header `Location` |
| 204 | berhasil, tanpa isi |
| 400 | request tidak bisa dibaca (JSON rusak, header wajib tidak ada) |
| 401 | belum login / sesi tidak sah |
| 403 | sudah login tapi tidak berhak |
| 404 | tidak ada, **atau** ada tapi bukan milik pemanggil (jangan bocorkan keberadaannya) |
| 409 | bentrok dengan keadaan sekarang (duplikat, sedang diproses) |
| 422 | isi request terbaca tapi tidak valid |
| 429 | terlalu banyak request; sertakan `Retry-After` [RFC 6585] |
| 500 | kesalahan server yang tidak terduga |
| 502 / 503 | layanan luar gagal / sedang tidak tersedia |

### 5. Format error seragam [RFC 9457]

Satu format error untuk semua endpoint. Proyek baru memakai RFC 9457
(`Content-Type: application/problem+json`):

```json
{
  "type": "https://example.com/problems/insufficient-balance",
  "title": "Saldo tidak cukup",
  "status": 409,
  "detail": "Saldo kurang untuk transaksi ini.",
  "code": "insufficient_balance",
  "errors": [{ "field": "amount", "code": "too_large" }]
}
```

- `code`: string tetap yang dibaca program. **Frontend menerjemahkan `code` ke kamus** (`manage-translations`),
  bukan menampilkan teks server mentah.
- `errors`: daftar kesalahan per field, untuk validasi form.
- **Tidak pernah** berisi stack trace, query SQL, nama tabel, path file, atau pesan library. [RFC 9457 §5]
- Error 500 selalu generik ke pemanggil; detailnya ke log server dengan ID request yang sama.

Proyek lama yang sudah memakai **satu** format lain secara seragam (mis. `{"error":{"code","message"}}`)
mencatatnya sebagai *konvensi*, asal memenuhi tiga syarat di atas: ada `code` tetap, ada kesalahan per field
untuk form, tidak membocorkan isi server. Kalau tidak memenuhi, itu *utang*. Mengganti format adalah
perubahan yang merusak (aturan 12): butuh ADR.

### 6. Validasi di server [OWASP Input Validation]

- Setiap input divalidasi **di server**: tipe, panjang, rentang, format, nilai yang diizinkan (allowlist).
  Validasi di frontend hanya untuk kenyamanan, bukan keamanan.
- **Tolak field yang tidak dikenal**, dan jangan pernah memetakan body request langsung ke baris database.
  Field seperti `role`, `balance`, `user_id` tidak boleh bisa diisi pemanggil. [OWASP Mass Assignment]
- Batasi ukuran body request.

### 7. Izin per endpoint [OWASP API Security Top 10]

- **Tolak secara bawaan.** Setiap endpoint menyatakan siapa yang boleh: publik, user login, admin.
- **Cek kepemilikan objek** di setiap akses berdasarkan ID: `GET /api-keys/123` harus memastikan kunci 123
  milik pemanggil. Ini celah API nomor 1 (BOLA). [OWASP API1:2023]
- Identitas pemanggil diambil dari sesi/token, **bukan** dari body atau query.

### 8. Bentuk data

| Jenis | Bentuk | Contoh |
|---|---|---|
| Nama field | `snake_case`, konsisten di semua endpoint [keputusan webcraft] | `created_at` |
| Uang | bilangan bulat satuan terkecil + mata uang | `{"amount": 150000, "currency": "IDR"}` |
| Waktu | string RFC 3339 dalam UTC | `"2026-10-29T07:00:00Z"` |
| ID | string | `"8412"` |
| Kosong | `null` atau field tidak ada, konsisten; daftar kosong = `[]` | |

- Response hanya berisi field yang dibutuhkan pemanggil. **Tidak pernah** mengembalikan hash sandi, rahasia,
  token internal, atau data milik user lain.

### 9. Daftar data

- **Wajib pagination.** `limit` dengan nilai bawaan (mis. 20) dan batas maksimum (mis. 100).
- Data yang terus bertambah tanpa batas (riwayat transaksi, log, feed) memakai **cursor**: response berisi
  `next_cursor`, kosong kalau habis. Offset besar makin lambat. [Use The Index, Luke: No offset]
- Offset (`page`, `per_page`) boleh untuk daftar yang kecil atau halaman admin bernomor, selama `limit`
  dibatasi. Kalau daftar itu nanti tumbuh dan melambat (diukur, `build-backend` aturan 12), pindah ke cursor.
- Filter dan urutan hanya dari daftar field yang diizinkan.

```json
{ "data": [ … ], "next_cursor": "eyJpZCI6ODQxMn0" }
```

### 10. Idempotency [IETF Idempotency-Key]

`POST` yang membuat transaksi uang, pesanan, atau mengirim pesan **wajib** menerima header `Idempotency-Key`
(string acak dari pemanggil, mis. UUID):

| Keadaan | Response |
|---|---|
| kunci baru | proses normal, simpan hasilnya bersama kunci |
| kunci sama, payload sama, sudah selesai | kembalikan hasil yang tersimpan, tanpa memproses ulang |
| kunci sama, request pertama masih diproses | 409 |
| kunci sama, payload berbeda | 422 |
| kunci tidak ada di endpoint yang mewajibkan | 400 |

Kunci punya masa berlaku (mis. 24 jam) yang ditulis di dokumentasi API. Cara menyimpannya: `build-backend`.

### 11. Rate limit

Endpoint login, daftar, reset sandi, pembayaran, dan API publik diberi batas. Melebihi batas → 429 +
`Retry-After`. Implementasi: `build-backend`.

### 12. Kompatibilitas

- **Aman:** menambah endpoint, menambah field response, menambah field request opsional.
- **Merusak:** menghapus/mengganti nama field, mengubah tipe atau arti, menjadikan field wajib. Lakukan
  bertahap: tambah yang baru → pemakai pindah → hapus yang lama, atau naikkan versi.
- Pemanggil wajib mengabaikan field yang tidak dikenalnya.

### 13. CORS

Hanya origin yang terdaftar. Tidak pernah `*` untuk endpoint yang memakai cookie/kredensial.

### 14. Koneksi panjang: SSE dan WebSocket [OWASP WebSocket Security]

- **SSE** (*Server-Sent Events*, mis. streaming jawaban AI) memakai aturan endpoint HTTP biasa: autentikasi,
  izin, rate limit. Stream dihentikan saat klien memutus koneksi, dan ada batas waktu/ukuran total.
- **WebSocket tidak dilindungi CORS.** Saat jabat tangan (*handshake*): cek header `Origin` terhadap daftar
  yang diizinkan, dan autentikasi (cookie sesi/kunci API). Token tidak dikirim lewat query string.
- Setiap pesan yang masuk divalidasi dan dicek izinnya seperti request biasa; jangan hanya mengandalkan cek di
  awal koneksi.
- Batasi ukuran pesan, jumlah pesan per detik, jumlah koneksi per user, dan putuskan koneksi yang menganggur.
- Pakai `wss://` (TLS). Sesi yang di-logout/dicabut juga memutus koneksi yang masih terbuka.

## Larangan

- `GET` yang mengubah data.
- Status 200 dengan isi `{"success": false}`.
- Pesan error berisi detail internal.
- Mengambil `user_id` atau peran dari body request.
- Daftar tanpa batas jumlah.
- Mengubah bentuk response tanpa memeriksa siapa pemakainya.

## Cek hasil

- [ ] Kontrak tertulis di spek fitur / dokumentasi API (`write-docs`).
- [ ] Test untuk: sukses, input tidak valid (422), tanpa login (401), milik orang lain (404/403).
- [ ] Error memakai format seragam; tidak ada detail internal.
- [ ] Endpoint daftar punya batas; endpoint transaksi menerima `Idempotency-Key`.

## Stack

- `references/go.md`: router, decode aman, helper error (backend Go)
- `references/nextjs.md`: satu klien HTTP, pemetaan error ke kamus (frontend Next.js)

## Sumber

- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110.html): method aman dan idempoten, status code.
- [RFC 9457: Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc9457.html).
- [RFC 6585](https://www.rfc-editor.org/rfc/rfc6585.html): 429 Too Many Requests.
- [IETF draft: The Idempotency-Key HTTP Header Field](https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/).
- [OWASP API Security Top 10 (2023)](https://owasp.org/API-Security/editions/2023/en/0x11-t10/).
- [OWASP Input Validation Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Input_Validation_Cheat_Sheet.html).
- [OWASP Mass Assignment Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Mass_Assignment_Cheat_Sheet.html).
- [OWASP REST Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/REST_Security_Cheat_Sheet.html).
- [OWASP WebSocket Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/WebSocket_Security_Cheat_Sheet.html).
- [Use The Index, Luke: Paging Through Results (no offset)](https://use-the-index-luke.com/no-offset).
