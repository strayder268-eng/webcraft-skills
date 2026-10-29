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

## Kerentanan dependensi

```sh
go install golang.org/x/vuln/cmd/govulncheck@latest
govulncheck ./...   # hanya melapor kerentanan pada kode yang benar-benar dipanggil
```

Jalankan di CI. `go.sum` di-commit.

## Sumber

- [Go: Vulnerability Management (govulncheck)](https://go.dev/doc/security/vuln/)
- [os/exec](https://pkg.go.dev/os/exec)
