# Log in with Telegram (OpenID Connect)

Telegram kini menyediakan login berbasis **OpenID Connect**. Widget lama yang memverifikasi `hash` dengan
token bot sudah dipindah ke arsip (`/widgets/login-legacy`); **proyek baru tidak memakainya**. Proyek lama yang
masih memakainya dicatat sebagai temuan audit dan dimigrasikan.

## Persiapan di @BotFather

- Buka bot → **Login Widget** → daftarkan **Allowed URLs**: origin situs (`https://example.com`) dan
  `redirect_uri` (`https://example.com/auth/telegram/callback`). Telegram hanya memproses URL yang terdaftar.
- Catat **Client ID** dan **Client Secret** → environment variable server (`structure-project`).
- Algoritma tanda tangan bawaan RS256 (bisa diganti di Login Widget → Advanced).

## Endpoint

| | URL |
|---|---|
| Discovery | `https://oauth.telegram.org/.well-known/openid-configuration` |
| Authorization | `https://oauth.telegram.org/auth` |
| Token | `https://oauth.telegram.org/token` |
| JWKS | `https://oauth.telegram.org/.well-known/jwks.json` |

Tidak ada endpoint UserInfo; data user ada di `id_token`. Matikan permintaan UserInfo kalau library mencobanya.

## Alur (server)

1. Server membuat `state`, `nonce`, dan `code_verifier` acak, menyimpannya di sesi sementara, lalu mengarahkan
   browser ke endpoint Authorization dengan `client_id`, `redirect_uri`, `response_type=code`,
   `scope=openid profile`, `state`, `code_challenge`, `code_challenge_method=S256` (dan `nonce`; SDK JavaScript
   Telegram mendukungnya, dan klaim `nonce` di `id_token` wajib dicek kalau dikirim).
2. Telegram mengarahkan kembali ke `redirect_uri` dengan `code` dan `state`. Cocokkan `state`.
3. Server POST ke endpoint Token: `grant_type=authorization_code`, `code`, `redirect_uri`, `client_id`,
   `code_verifier`, dengan header `Authorization: Basic base64(client_id:client_secret)`.
4. Verifikasi `id_token`: tanda tangan dengan kunci JWKS, `iss = https://oauth.telegram.org`,
   `aud` = Bot/Client ID, `exp` belum lewat, `nonce` cocok (kalau dikirim).
5. Identitas user = `sub`. Simpan di tabel identitas sebagai `(provider='telegram', subject=sub)`.

## Scope

| Scope | Isi |
|---|---|
| `openid` | wajib: `sub`, `iss`, `iat`, `exp` |
| `profile` | `name`, `preferred_username`, `picture` |
| `phone` | nomor telepon, butuh persetujuan user |
| `telegram:bot_access` | bot boleh mengirim pesan langsung ke user setelah login |

Minta scope seperlunya.

## Varian JavaScript (popup)

`Telegram.Login.auth({ client_id, scope, nonce }, callback)` mengembalikan `id_token` ke browser. Browser
**mengirim `id_token` ke server**, dan server memverifikasinya persis seperti langkah 4. Data `user` yang sudah
didekode di browser tidak dipercaya. Halaman dengan popup tidak boleh mengirim
`Cross-Origin-Opener-Policy: same-origin` (pakai `same-origin-allow-popups`).

## Sumber

- [Telegram: Log in with Telegram](https://core.telegram.org/widgets/login)
