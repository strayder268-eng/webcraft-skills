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

## Input berbahaya (SQL injection)

```go
func TestSearchTreatsInputAsText(t *testing.T) {
    pool := testDB(t)
    // data milik test ini saja
    seed(t, pool, "Kopi Arabika", "Teh 100% Melati", "Gula_Aren")
    before := countRows(t, pool, "products")

    cases := []struct {
        q    string
        want []string
    }{
        {"' OR '1'='1", nil},                  // bukan "semua baris"
        {"'; DROP TABLE products; --", nil},
        {"%", []string{"Teh 100% Melati"}},     // % dicari sebagai huruf, bukan wildcard
        {"_", []string{"Gula_Aren"}},
        {`\`, nil},
    }
    for _, tc := range cases {
        t.Run(tc.q, func(t *testing.T) {
            got, err := store.Search(ctx, tc.q)
            if err != nil { t.Fatal(err) }
            if !reflect.DeepEqual(names(got), tc.want) { t.Fatalf("got %v, want %v", names(got), tc.want) }
        })
    }
    if after := countRows(t, pool, "products"); after != before {
        t.Fatalf("jumlah baris berubah: %d → %d", before, after)
    }
}

func TestListRejectsUnknownSort(t *testing.T) {
    for _, s := range []string{"name; DROP TABLE users", "password_hash", "1"} {
        rec := get(t, "/api/v1/products?sort="+url.QueryEscape(s))
        if rec.Code != http.StatusUnprocessableEntity { t.Fatalf("sort=%q: status %d, want 422", s, rec.Code) }
    }
}
```

## Jumlah query (mencegah N+1)

Memakai penghitung query dari `build-backend/references/go.md` (pool test dibuat dengan
`cfg.ConnConfig.Tracer = db.CountingTracer{}`):

```go
func TestListOrdersQueryCountIsConstant(t *testing.T) {
    pool := testDB(t)
    svc := orders.NewService(pool)
    userID := seedUserWithOrders(t, pool, 50)

    count := func(limit int) int64 {
        ctx, stats := db.WithQueryStats(context.Background())
        if _, err := svc.List(ctx, userID, limit); err != nil {
            t.Fatal(err)
        }
        return stats.Count.Load()
    }

    small, large := count(5), count(50)
    if small != large {
        t.Fatalf("jumlah query bertambah mengikuti baris (%d untuk 5, %d untuk 50): N+1", small, large)
    }
    if large > 3 {
        t.Fatalf("List memakai %d query, batas 3", large)
    }
}
```

Contoh hasil nyata pada 100 baris: versi N+1 = 101 query (18,5 ms di database lokal), versi `ANY($1)` = 2
query (1,0 ms), versi `JOIN` = 1 query (0,7 ms). Di server dengan jeda jaringan ke database, selisihnya jauh
lebih besar.

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
