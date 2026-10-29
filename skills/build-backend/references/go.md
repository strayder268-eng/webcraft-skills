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

## Goroutine

Setiap goroutine latar punya cara berhenti (`ctx.Done()`), dan `main` menunggunya selesai saat shutdown.

## Sumber

- [log/slog](https://pkg.go.dev/log/slog)
- [errors: Is, As, wrapping](https://pkg.go.dev/errors)
- [net/http: Server.Shutdown](https://pkg.go.dev/net/http#Server.Shutdown)
- [os/signal: NotifyContext](https://pkg.go.dev/os/signal#NotifyContext)
