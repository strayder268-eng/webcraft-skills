# Auth di frontend Next.js (backend terpisah)

Backend yang memegang sesi dan izin. Frontend hanya menampilkan dan mengarahkan.

## Cookie

- Cookie sesi `HttpOnly`: JavaScript di browser tidak bisa (dan tidak perlu) membacanya.
- Semua request ke API memakai `credentials: 'include'` (`design-api/references/nextjs.md`).
- Frontend tidak pernah menyimpan token di `localStorage`/`sessionStorage`.

## Siapa yang sedang login

Endpoint backend `GET /api/v1/me` mengembalikan user dan perannya (401 kalau belum login). Frontend memakai
jawaban ini untuk menampilkan nama, menu, dan peran.

## Proteksi halaman

- `src/proxy.ts` boleh melakukan **cek optimistis**: kalau cookie sesi tidak ada sama sekali, arahkan ke
  `/login`. Proxy berjalan di setiap route, jadi jangan memanggil database/API di sana. [Next.js: Authentication]
- Cek sesungguhnya tetap di backend: API mengembalikan 401/403, dan halaman menangani itu (arahkan ke login /
  tampilkan "tidak punya akses").
- Menyembunyikan menu admin hanya soal tampilan; endpoint admin tetap dijaga backend.

## Halaman login

- Form mengikuti `build-ui` (label, `autocomplete="email"`, `autocomplete="current-password"`,
  `autocomplete="one-time-code"` untuk 2FA).
- Tombol "Masuk dengan Google/Telegram" mengarah ke endpoint backend yang memulai alur OIDC
  (`/api/v1/auth/google/start`), bukan memproses OAuth di browser.
- Setelah login, kembali ke halaman asal hanya kalau alamatnya **path internal**: diawali satu `/`, bukan
  `//` atau `/\` (keduanya bisa diartikan browser sebagai domain lain). Selain itu arahkan ke halaman bawaan.
  Tidak pernah ke URL domain lain (open redirect).

## Sumber

- [Next.js: Authentication](https://nextjs.org/docs/app/guides/authentication)
- [OWASP Unvalidated Redirects and Forwards](https://cheatsheetseries.owasp.org/cheatsheets/Unvalidated_Redirects_and_Forwards_Cheat_Sheet.html)
