# Database dari Go

Rekomendasi untuk PostgreSQL: driver **pgx v5** dan alat migrasi **goose** (file SQL biasa). Kalau proyek
sudah memakai yang lain, ikuti yang ada. Menambah salah satunya ke proyek dicatat sebagai ADR.

## Pool

```go
// internal/platform/db
func Open(ctx context.Context, url string) (*pgxpool.Pool, error) {
    cfg, err := pgxpool.ParseConfig(url)
    if err != nil {
        return nil, err
    }
    cfg.MaxConns = 10 // dibatasi; sesuaikan dengan kapasitas database
    pool, err := pgxpool.NewWithConfig(ctx, cfg)
    if err != nil {
        return nil, err
    }
    if err := pool.Ping(ctx); err != nil { // gagal cepat saat start
        pool.Close()
        return nil, err
    }
    return pool, nil
}
```

Satu pool untuk seluruh aplikasi, dibuat di `main`, dioper ke setiap `store`.

## Query

- Selalu pakai `ctx` dari request (`r.Context()`), supaya query berhenti kalau user membatalkan.
- Parameter `$1, $2`, tidak pernah `fmt.Sprintf` untuk menyusun SQL dari input.
- Banyak ID sekaligus (hindari N+1):

```go
rows, err := pool.Query(ctx, `SELECT id, name FROM users WHERE id = ANY($1)`, ids) // ids []int64
```

## Transaksi

```go
err := pgx.BeginFunc(ctx, pool, func(tx pgx.Tx) error {
    if _, err := tx.Exec(ctx, `INSERT INTO ledger_entries (account_id, amount, kind, idempotency_key)
        VALUES ($1, $2, $3, $4)`, acc, amount, kind, key); err != nil {
        return err
    }
    _, err := tx.Exec(ctx, `UPDATE accounts SET balance = balance + $2 WHERE id = $1`, acc, amount)
    return err
}) // commit kalau fungsi mengembalikan nil, rollback kalau error
```

Tidak ada panggilan HTTP/email di dalam fungsi transaksi.

## Mengenali pelanggaran constraint

```go
var pgErr *pgconn.PgError
if errors.As(err, &pgErr) {
    switch pgErr.Code {
    case "23505": // unique_violation → duplikat / kunci idempotency sudah dipakai → 409 atau hasil lama
    case "23514": // check_violation → mis. saldo jadi negatif → 409 insufficient_balance
    case "23503": // foreign_key_violation → data acuan tidak ada → 422
    }
}
```

## Migrasi dengan goose

```
api/migrations/
  00001_create_users.sql
  00002_create_api_keys.sql
```

```sql
-- +goose Up
CREATE TABLE …;

-- +goose Down
DROP TABLE …;
```

File yang berisi `CREATE INDEX CONCURRENTLY` diberi `-- +goose NO TRANSACTION` di baris pertama.

Migrasi dijalankan sebagai **langkah terpisah sebelum deploy** (mis. `cmd/migrate`), bukan otomatis setiap
aplikasi start, supaya beberapa instance tidak menjalankannya bersamaan dan kegagalan migrasi tidak
menjatuhkan aplikasi.

## Sumber

- [pgx v5](https://pkg.go.dev/github.com/jackc/pgx/v5) dan [pgxpool](https://pkg.go.dev/github.com/jackc/pgx/v5/pgxpool).
- [PostgreSQL: Error Codes](https://www.postgresql.org/docs/current/errcodes-appendix.html).
- [goose](https://github.com/pressly/goose).
