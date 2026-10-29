# Test di Go

## Test tabel

```go
func TestValidateTopUp(t *testing.T) {
    cases := []struct {
        name    string
        amount  int64
        wantErr error
    }{
        {"nominal normal", 50_000, nil},
        {"nol ditolak", 0, ErrInvalidAmount},
        {"negatif ditolak", -1, ErrInvalidAmount},
        {"di atas batas ditolak", 100_000_001, ErrAmountTooLarge},
    }
    for _, tc := range cases {
        t.Run(tc.name, func(t *testing.T) {
            err := ValidateTopUp(tc.amount)
            if !errors.Is(err, tc.wantErr) {
                t.Fatalf("got %v, want %v", err, tc.wantErr)
            }
        })
    }
}
```

## Handler lewat httptest

```go
req := httptest.NewRequest(http.MethodPost, "/api/v1/api-keys", strings.NewReader(`{"name":"ci"}`))
req.AddCookie(sessionCookieFor(t, userID))
rec := httptest.NewRecorder()
router.ServeHTTP(rec, req)
if rec.Code != http.StatusCreated { t.Fatalf("status %d: %s", rec.Code, rec.Body) }
```

## Layanan luar tiruan

```go
gw := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusServiceUnavailable) // uji jalur gagal
}))
defer gw.Close()
client := payment.NewClient(gw.URL, secret) // URL layanan diambil dari config, jadi bisa diganti di test
```

## Database test

Container sendiri, bukan database pengembangan:

```sh
docker run -d --name test-pg -e POSTGRES_PASSWORD=t -e POSTGRES_USER=t -e POSTGRES_DB=t \
  -p 127.0.0.1:55432:5432 postgres:16-alpine
# jalankan migrasi ke database ini, lalu:
DATABASE_URL=postgres://t:t@127.0.0.1:55432/t?sslmode=disable go test -race -count=1 ./...
docker rm -f test-pg
```

Di dalam test:

```go
func testDB(t *testing.T) *pgxpool.Pool {
    url := os.Getenv("DATABASE_URL")
    if url == "" {
        t.Skip("DATABASE_URL tidak diset") // di CI wajib diset, jadi test ini tidak pernah dilewati di sana
    }
    pool, err := pgxpool.New(context.Background(), url)
    if err != nil { t.Fatal(err) }
    t.Cleanup(pool.Close)
    return pool
}

// data unik per test + bersih-bersih
email := fmt.Sprintf("u-%d@test.local", time.Now().UnixNano())
t.Cleanup(func() { pool.Exec(context.Background(), `DELETE FROM users WHERE email = $1`, email) })
```

## Bersamaan

```go
var wg sync.WaitGroup
errs := make(chan error, 10)
for range 10 {
    wg.Add(1)
    go func() { defer wg.Done(); errs <- svc.Charge(ctx, userID, 10_000) }() // saldo awal 50.000
}
wg.Wait(); close(errs)
// harus tepat 5 berhasil, 5 ErrInsufficientBalance, saldo akhir 0
```

## Perintah

```sh
go test -race -count=1 ./...   # -count=1: tanpa cache hasil; -race: deteksi data race
```

## Sumber

- [Go: testing](https://pkg.go.dev/testing)
- [net/http/httptest](https://pkg.go.dev/net/http/httptest)
- [Go: Data Race Detector](https://go.dev/doc/articles/race_detector)
