# Keamanan di Go

## Header untuk API JSON

```go
func SecurityHeaders(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        h := w.Header()
        h.Set("X-Content-Type-Options", "nosniff")
        h.Set("Referrer-Policy", "strict-origin-when-cross-origin")
        h.Set("Content-Security-Policy", "default-src 'none'; frame-ancestors 'none'") // API tidak menyajikan halaman
        h.Set("Cache-Control", "no-store") // response berisi data user tidak disimpan cache bersama
        next.ServeHTTP(w, r)
    })
}
```

`Cache-Control: no-store` untuk response pribadi; endpoint publik yang memang boleh di-cache mengaturnya sendiri.

## Menjalankan perintah sistem

Hindari. Kalau terpaksa: `exec.CommandContext(ctx, "program", arg1, arg2)` dengan argumen terpisah, **tidak
pernah** `sh -c` dengan string yang mengandung input user.

## Membuka file tanpa path traversal (Go 1.24+)

```go
root, err := os.OpenRoot("/data/uploads") // dibuka sekali saat start
if err != nil { return err }
defer root.Close()

f, err := root.Open(name) // "../x", "/etc/passwd", symlink keluar folder → ditolak
```

Tetap utamakan mengambil file lewat ID dari database; `os.Root` adalah pengaman terakhir. [os.Root]

## Membuka arsip zip dari user

```go
const maxFiles, maxTotal = 1000, 200 << 20 // 1000 file, 200 MB setelah dibuka
zr, err := zip.OpenReader(path)
if err != nil { return err }
defer zr.Close()
if len(zr.File) > maxFiles { return ErrArchiveTooLarge }
var total int64
for _, f := range zr.File {
    if !filepath.IsLocal(f.Name) { return ErrBadArchive } // tolak "../", path absolut (zip slip)
    rc, err := f.Open()
    if err != nil { return err }
    n, err := io.Copy(dst(f.Name), io.LimitReader(rc, maxTotal-total+1)) // hitung ukuran asli, bukan header
    rc.Close()
    if err != nil { return err }
    if total += n; total > maxTotal { return ErrArchiveTooLarge }
}
```

Tulis hasil ekstrak lewat `os.Root` folder tujuan juga.

## Verifikasi token anti-bot (Turnstile)

```go
form := url.Values{"secret": {cfg.TurnstileSecret}, "response": {token}, "remoteip": {clientIP}}
resp, err := httpClient.PostForm("https://challenges.cloudflare.com/turnstile/v0/siteverify", form)
// baca JSON: success harus true; cek juga hostname/action sesuai halaman. Gagal/timeout → tolak (gagal tertutup).
```

## Kerentanan dependensi

```sh
go install golang.org/x/vuln/cmd/govulncheck@<versi>   # versi dikunci
govulncheck ./...   # hanya melapor kerentanan pada kode yang benar-benar dipanggil
```

Jalankan di CI. `go.sum` di-commit.

## Sumber

- [Go: Vulnerability Management (govulncheck)](https://go.dev/doc/security/vuln/)
- [os/exec](https://pkg.go.dev/os/exec)
- [os.Root](https://pkg.go.dev/os#Root) dan [Go blog: Traversal-resistant file APIs](https://go.dev/blog/osroot)
- [path/filepath: IsLocal](https://pkg.go.dev/path/filepath#IsLocal)
- [Cloudflare Turnstile: Server-side validation](https://developers.cloudflare.com/turnstile/get-started/server-side-validation/)
