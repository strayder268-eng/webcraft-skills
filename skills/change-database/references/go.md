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

## Urutan dari input (allowlist)

```go
// kunci dari user → potongan SQL tetap; tidak ada teks user yang masuk ke SQL
var sortColumns = map[string]string{
    "created": "created_at",
    "name":    "name",
}

func orderBy(sort, dir string) (string, error) {
    col, ok := sortColumns[sort]
    if !ok {
        return "", ErrInvalidSort // → 422
    }
    d := "ASC"
    if dir == "desc" {
        d = "DESC"
    }
    return " ORDER BY " + col + " " + d + ", id " + d, nil
}
```

## Pencarian LIKE

```go
var likeEscaper = strings.NewReplacer(`\`, `\\`, `%`, `\%`, `_`, `\_`)

q := strings.TrimSpace(input)
if len(q) > 100 {
    return ErrSearchTooLong // → 422
}
rows, err := pool.Query(ctx,
    `SELECT id, name FROM products WHERE name ILIKE '%' || $1 || '%' ESCAPE '\' LIMIT 50`,
    likeEscaper.Replace(q))
```

## Pengecekan otomatis

[gosec](https://github.com/securego/gosec) di CI, minimal aturan **G201** (SQL disusun dengan format string)
dan **G202** (SQL disusun dengan penyambungan string):

```sh
go run github.com/securego/gosec/v2/cmd/gosec@<versi> -include=G201,G202 ./...
```

Potongan SQL tetap dari allowlist seperti `orderBy` di atas bisa ikut ditandai; beri komentar `// #nosec G202`
**beserta alasannya** di baris itu saja, jangan mematikan aturannya.

## Enkripsi kolom rahasia (AES-256-GCM)

Pustaka standar Go saja. Format tersimpan: `<idKunci>:<base64(nonce|ciphertext)>`.

```go
// internal/platform/secretbox
type Box struct {
    current string                 // ID kunci untuk enkripsi baru
    keys    map[string]cipher.AEAD // semua kunci yang masih bisa membuka data lama
}

func NewBox(current string, rawKeys map[string][]byte) (*Box, error) {
    b := &Box{current: current, keys: map[string]cipher.AEAD{}}
    for id, k := range rawKeys {
        if len(k) != 32 {
            return nil, errors.New("kunci harus 32 byte (AES-256)")
        }
        block, err := aes.NewCipher(k)
        if err != nil {
            return nil, err
        }
        gcm, err := cipher.NewGCM(block)
        if err != nil {
            return nil, err
        }
        b.keys[id] = gcm
    }
    if _, ok := b.keys[current]; !ok {
        return nil, errors.New("kunci aktif tidak ada")
    }
    return b, nil
}

// aad mengikat nilai ke tempatnya, mis. "provider_keys.api_key:42".
func (b *Box) Seal(plain, aad []byte) (string, error) {
    gcm := b.keys[b.current]
    nonce := make([]byte, gcm.NonceSize())
    if _, err := rand.Read(nonce); err != nil { // crypto/rand
        return "", err
    }
    out := gcm.Seal(nonce, nonce, plain, aad)
    return b.current + ":" + base64.RawStdEncoding.EncodeToString(out), nil
}

func (b *Box) Open(stored string, aad []byte) ([]byte, error) {
    id, enc, ok := strings.Cut(stored, ":")
    gcm, known := b.keys[id]
    if !ok || !known {
        return nil, errors.New("format/kunci tidak dikenal")
    }
    raw, err := base64.RawStdEncoding.DecodeString(enc)
    if err != nil || len(raw) < gcm.NonceSize() {
        return nil, errors.New("data rusak")
    }
    return gcm.Open(nil, raw[:gcm.NonceSize()], raw[gcm.NonceSize():], aad)
}
```

- Kunci dibaca dari env saat start (mis. `SECRET_KEYS=k1:<base64>,k2:<base64>`, `SECRET_KEY_CURRENT=k2`).
- Rotasi: tambah kunci baru sebagai `current`, kunci lama tetap ada untuk membuka data lama; pekerjaan latar
  mengenkripsi ulang baris lama; setelah selesai, kunci lama dihapus.
- Membuka dengan `aad` yang berbeda (nilai dipindah ke baris lain) gagal dengan error autentikasi.

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
- [gosec](https://github.com/securego/gosec): aturan G201, G202.
- [crypto/cipher: NewGCM](https://pkg.go.dev/crypto/cipher#NewGCM).
