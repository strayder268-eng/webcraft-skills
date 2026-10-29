# Backend Go

## Error bernama

```go
// internal/billing/service.go
var (
    ErrInsufficientBalance = errors.New("insufficient balance")
    ErrNotFound            = errors.New("not found")
)

func (s *Service) Charge(ctx context.Context, userID int64, amount int64) error {
    if err := s.store.Debit(ctx, userID, amount); err != nil {
        return fmt.Errorf("debit user %d: %w", userID, err) // tambah konteks, pertahankan error asli
    }
    return nil
}
```

```go
// handler: petakan sekali
switch {
case errors.Is(err, billing.ErrInsufficientBalance):
    writeProblem(w, Problem{Status: 409, Title: "Saldo tidak cukup", Code: "insufficient_balance"})
case errors.Is(err, billing.ErrNotFound):
    writeProblem(w, Problem{Status: 404, Title: "Tidak ditemukan", Code: "not_found"})
default:
    slog.ErrorContext(ctx, "charge gagal", "err", err)
    writeProblem(w, Problem{Status: 500, Title: "Terjadi kesalahan", Code: "internal_error"})
}
```

## Log dengan slog

```go
logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{Level: slog.LevelInfo}))
slog.SetDefault(logger)
```

Middleware menaruh ID request di `context` dan menambahkannya ke setiap log request itu. Pakai
`slog.InfoContext(ctx, …)` / `slog.ErrorContext(ctx, …)` supaya ID ikut tercatat.

## Klien HTTP ke layanan luar

```go
// dibuat sekali, dipakai ulang
var paymentClient = &http.Client{Timeout: 10 * time.Second}

req, _ := http.NewRequestWithContext(ctx, http.MethodPost, url, body)
resp, err := paymentClient.Do(req)
if err != nil { /* timeout / jaringan → boleh retry kalau idempoten */ }
defer resp.Body.Close()
```

`http.DefaultClient` tidak punya timeout; jangan dipakai untuk layanan luar.

## Server dan shutdown

```go
srv := &http.Server{
    Addr:              ":8080",
    Handler:           handler,
    ReadHeaderTimeout: 5 * time.Second,
    ReadTimeout:       15 * time.Second,
    WriteTimeout:      30 * time.Second, // perbesar/atur per route untuk streaming
    IdleTimeout:       60 * time.Second,
}

ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
defer stop()

go func() {
    if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
        slog.Error("server berhenti", "err", err)
        os.Exit(1)
    }
}()

<-ctx.Done()
shutdownCtx, cancel := context.WithTimeout(context.Background(), 20*time.Second)
defer cancel()
_ = srv.Shutdown(shutdownCtx) // berhenti terima request baru, tunggu yang berjalan
pool.Close()
```

## Mengukur query per request (pgx)

```go
// internal/platform/db: hitung query dan waktu database untuk setiap request
type statsKey struct{}
type startKey struct{}

type QueryStats struct{ Count, DBns atomic.Int64 }

func WithQueryStats(ctx context.Context) (context.Context, *QueryStats) {
    s := &QueryStats{}
    return context.WithValue(ctx, statsKey{}, s), s
}

func StatsFrom(ctx context.Context) *QueryStats {
    s, _ := ctx.Value(statsKey{}).(*QueryStats)
    return s
}

type CountingTracer struct{}

func (CountingTracer) TraceQueryStart(ctx context.Context, _ *pgx.Conn, _ pgx.TraceQueryStartData) context.Context {
    return context.WithValue(ctx, startKey{}, time.Now())
}

func (CountingTracer) TraceQueryEnd(ctx context.Context, _ *pgx.Conn, _ pgx.TraceQueryEndData) {
    s := StatsFrom(ctx)
    if s == nil {
        return
    }
    s.Count.Add(1)
    if t, ok := ctx.Value(startKey{}).(time.Time); ok {
        s.DBns.Add(int64(time.Since(t)))
    }
}

// saat membuat pool: cfg.ConnConfig.Tracer = CountingTracer{}
```

Query di dalam `pgx.Batch` memakai `BatchTracer` terpisah; tambahkan kalau proyek memakai batch.

## Middleware: log request + Server-Timing

```go
type timingWriter struct {
    http.ResponseWriter
    start  time.Time
    stats  *QueryStats
    expose bool
    status int
}

func (w *timingWriter) WriteHeader(code int) {
    if w.status == 0 {
        w.status = code
        if w.expose { // header harus dipasang sebelum response mulai dikirim
            w.Header().Set("Server-Timing", fmt.Sprintf("db;dur=%.1f, total;dur=%.1f",
                float64(w.stats.DBns.Load())/1e6, float64(time.Since(w.start))/1e6))
        }
    }
    w.ResponseWriter.WriteHeader(code)
}

func (w *timingWriter) Write(b []byte) (int, error) {
    if w.status == 0 {
        w.WriteHeader(http.StatusOK)
    }
    return w.ResponseWriter.Write(b)
}

// supaya http.NewResponseController(w).Flush() tetap jalan untuk streaming (SSE)
func (w *timingWriter) Unwrap() http.ResponseWriter { return w.ResponseWriter }

func Measure(exposeTiming bool, next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        ctx, stats := WithQueryStats(r.Context())
        tw := &timingWriter{ResponseWriter: w, start: time.Now(), stats: stats, expose: exposeTiming}
        r2 := r.WithContext(ctx)
        next.ServeHTTP(tw, r2)
        slog.InfoContext(ctx, "request",
            // r2.Pattern (Go 1.23+) diisi ServeMux pada request yang ia terima, jadi baca dari r2, bukan r.
            // Pola route ("GET /api/v1/orders/{id}") dipakai, bukan URL yang berisi ID.
            "method", r.Method, "path", r2.Pattern, "status", tw.status,
            "dur_ms", time.Since(tw.start).Milliseconds(),
            "queries", stats.Count.Load(),
            "db_ms", time.Duration(stats.DBns.Load()).Milliseconds())
    })
}
```

`Measure` dipasang langsung membungkus `mux`; middleware lain yang menyalin request di antaranya membuat
`Pattern` kosong. `exposeTiming` hanya `true` di pengembangan (atau untuk admin). Contoh log:
`{"msg":"request","path":"GET /api/v1/orders","status":200,"dur_ms":14,"queries":3,"db_ms":4}`.

## pprof di port internal

```go
// jangan import _ "net/http/pprof" di server publik: ia mendaftar ke DefaultServeMux
dbg := http.NewServeMux()
dbg.HandleFunc("/debug/pprof/", pprof.Index)
dbg.HandleFunc("/debug/pprof/profile", pprof.Profile)
dbg.HandleFunc("/debug/pprof/trace", pprof.Trace)
go http.ListenAndServe("127.0.0.1:6060", dbg) // hanya dari dalam container/server
```

```sh
go tool pprof http://127.0.0.1:6060/debug/pprof/profile?seconds=30   # CPU
go tool pprof http://127.0.0.1:6060/debug/pprof/heap                  # memori
```

## Goroutine

Setiap goroutine latar punya cara berhenti (`ctx.Done()`), dan `main` menunggunya selesai saat shutdown.

## Sumber

- [log/slog](https://pkg.go.dev/log/slog)
- [errors: Is, As, wrapping](https://pkg.go.dev/errors)
- [net/http: Server.Shutdown](https://pkg.go.dev/net/http#Server.Shutdown)
- [os/signal: NotifyContext](https://pkg.go.dev/os/signal#NotifyContext)
- [pgx: QueryTracer](https://pkg.go.dev/github.com/jackc/pgx/v5#QueryTracer)
- [net/http/pprof](https://pkg.go.dev/net/http/pprof)
- [net/http: ResponseController](https://pkg.go.dev/net/http#ResponseController)
- [MDN: Server-Timing](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Server-Timing)
