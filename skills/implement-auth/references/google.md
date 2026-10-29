# Sign in with Google (OpenID Connect)

## Persiapan

- Google Cloud Console → buat OAuth client tipe **Web application**.
- **Authorized redirect URIs**: persis `https://example.com/auth/google/callback` (dan versi lokal untuk
  pengembangan, terpisah).
- Client ID dan Client Secret → environment variable server.
- Isi layar persetujuan (nama aplikasi, domain, kebijakan privasi).

## Alur (server)

Discovery: `https://accounts.google.com/.well-known/openid-configuration` (ambil endpoint dari sini, jangan
ditulis manual).

1. Arahkan ke authorization endpoint dengan `response_type=code`, `scope=openid email profile`, `state`,
   `nonce`, `code_challenge` (S256).
2. Callback: cocokkan `state`, tukar `code` + `code_verifier` di server.
3. Verifikasi `id_token`: tanda tangan (JWKS Google), `iss` = `https://accounts.google.com` atau
   `accounts.google.com`, `aud` = client ID kita, `exp`, `nonce`.
4. Identitas = `sub`. Email dipakai hanya kalau `email_verified` bernilai `true`.
5. Klaim `hd` (domain Google Workspace) bukan pengaman; kalau membatasi domain, cek juga email-nya.

## Sumber

- [Google Identity: OpenID Connect](https://developers.google.com/identity/openid-connect/openid-connect)
