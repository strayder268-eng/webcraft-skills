# Auth di Go

Pustaka: `golang.org/x/crypto/argon2`, `golang.org/x/oauth2`, `github.com/coreos/go-oidc/v3/oidc`.
Menambahkannya ke proyek dicatat di ADR metode login.

## Hash sandi (Argon2id)

```go
const (
    argonTime    = 2
    argonMemory  = 19 * 1024 // KiB = 19 MiB (minimum OWASP)
    argonThreads = 1
    argonKeyLen  = 32
)

func HashPassword(pw string) (string, error) {
    salt := make([]byte, 16)
    if _, err := rand.Read(salt); err != nil {
        return "", err
    }
    key := argon2.IDKey([]byte(pw), salt, argonTime, argonMemory, argonThreads, argonKeyLen)
    return fmt.Sprintf("$argon2id$v=19$m=%d,t=%d,p=%d$%s$%s", argonMemory, argonTime, argonThreads,
        base64.RawStdEncoding.EncodeToString(salt), base64.RawStdEncoding.EncodeToString(key)), nil
}

// VerifyPassword membaca parameter dari string tersimpan (supaya parameter bisa dinaikkan nanti),
// menghitung ulang, lalu membandingkan dengan subtle.ConstantTimeCompare.
```

## Sesi

```go
func newSessionID() (raw string, hash []byte, err error) {
    b := make([]byte, 32) // 256 bit
    if _, err = rand.Read(b); err != nil {
        return
    }
    raw = base64.RawURLEncoding.EncodeToString(b)
    h := sha256.Sum256([]byte(raw))
    return raw, h[:], nil // simpan hash di DB, kirim raw ke cookie
}

http.SetCookie(w, &http.Cookie{
    Name:     "__Host-session",
    Value:    raw,
    Path:     "/",
    HttpOnly: true,
    Secure:   true,
    SameSite: http.SameSiteLaxMode, // Strict untuk admin
    MaxAge:   int((30 * 24 * time.Hour).Seconds()),
})
```

Middleware auth: baca cookie → hash → cari sesi yang belum kedaluwarsa → perbarui waktu aktif terakhir →
taruh user di `context`. Tidak ketemu → anonim (endpoint yang butuh login mengembalikan 401).

## Cek Origin (CSRF)

```go
func RequireSameOrigin(allowed map[string]bool, next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        switch r.Method {
        case http.MethodGet, http.MethodHead, http.MethodOptions:
        default:
            if !allowed[r.Header.Get("Origin")] {
                writeProblem(w, Problem{Status: 403, Title: "Origin tidak diizinkan", Code: "bad_origin"})
                return
            }
        }
        next.ServeHTTP(w, r)
    })
}
```

## OIDC dengan PKCE

```go
provider, err := oidc.NewProvider(ctx, "https://accounts.google.com") // atau https://oauth.telegram.org
cfg := oauth2.Config{
    ClientID:     env.ClientID,
    ClientSecret: env.ClientSecret,
    Endpoint:     provider.Endpoint(),
    RedirectURL:  env.RedirectURL,
    Scopes:       []string{oidc.ScopeOpenID, "profile"},
}

// mulai
state, nonce, verifier := randomString(), randomString(), oauth2.GenerateVerifier()
// simpan ketiganya di cookie sementara terenkripsi / sesi pra-login (berlaku ±10 menit)
http.Redirect(w, r, cfg.AuthCodeURL(state, oauth2.S256ChallengeOption(verifier), oidc.Nonce(nonce)), http.StatusFound)

// callback
if r.URL.Query().Get("state") != saved.State { /* tolak */ }
tok, err := cfg.Exchange(ctx, r.URL.Query().Get("code"), oauth2.VerifierOption(verifier))
rawID, _ := tok.Extra("id_token").(string)
idt, err := provider.Verifier(&oidc.Config{ClientID: env.ClientID}).Verify(ctx, rawID) // tanda tangan, iss, aud, exp
if idt.Nonce != saved.Nonce { /* tolak */ }
// identitas = idt.Subject
```

## Sumber

- [golang.org/x/crypto/argon2](https://pkg.go.dev/golang.org/x/crypto/argon2)
- [golang.org/x/oauth2: GenerateVerifier, S256ChallengeOption](https://pkg.go.dev/golang.org/x/oauth2)
- [go-oidc v3](https://pkg.go.dev/github.com/coreos/go-oidc/v3/oidc)
- [MDN: Set-Cookie (cookie prefixes)](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Set-Cookie)
