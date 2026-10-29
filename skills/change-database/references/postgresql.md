# PostgreSQL

Contoh untuk PostgreSQL 13+. Fitur yang butuh versi lebih baru diberi tanda.

## Kerangka tabel

```sql
CREATE TABLE api_keys (
  id          bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  user_id     bigint NOT NULL REFERENCES users (id) ON DELETE RESTRICT,
  name        text   NOT NULL CHECK (name <> '' AND char_length(name) <= 100),
  status      text   NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'revoked')),
  created_at  timestamptz NOT NULL DEFAULT now(),
  updated_at  timestamptz NOT NULL DEFAULT now()
);

CREATE INDEX api_keys_user_id_idx ON api_keys (user_id);
```

- `updated_at` diisi `now()` di setiap `UPDATE` oleh aplikasi.
- UUID: `gen_random_uuid()` (bawaan sejak 13); `uuidv7()` yang berurutan waktu tersedia sejak 18.

## Migrasi tanpa mengunci tabel lama

Awali setiap migrasi dengan batas tunggu kunci, supaya migrasi gagal cepat daripada membuat antrean request:

```sql
SET lock_timeout = '5s';
```

| Perubahan | Cara aman |
|---|---|
| Tambah kolom | boleh kosong, atau dengan `DEFAULT` nilai tetap: cepat sejak PostgreSQL 11 |
| Tambah index | `CREATE INDEX CONCURRENTLY …`, **di luar transaksi** (alat migrasi harus diatur tanpa transaksi untuk file ini) |
| Jadikan kolom `NOT NULL` | `ADD CONSTRAINT x CHECK (kol IS NOT NULL) NOT VALID;` → `VALIDATE CONSTRAINT x;` → `ALTER COLUMN kol SET NOT NULL;` (12+ memakai CHECK tadi, tanpa memindai ulang) → `DROP CONSTRAINT x;` |
| Tambah foreign key | `ADD CONSTRAINT … FOREIGN KEY … NOT VALID;` lalu `VALIDATE CONSTRAINT …;` di migrasi berikutnya |
| Isi data lama (backfill) | bertahap per ±1000 baris, bukan satu `UPDATE` raksasa |
| Ganti nama / tipe / hapus kolom | expand → migrate → contract (lihat SKILL.md) |

## Pagination keyset

```sql
-- halaman pertama
SELECT id, name, created_at
FROM api_keys
WHERE user_id = $1
ORDER BY created_at DESC, id DESC
LIMIT 21;                       -- ambil limit + 1 untuk tahu masih ada halaman berikutnya

-- halaman berikutnya: cursor = (created_at, id) baris terakhir
SELECT id, name, created_at
FROM api_keys
WHERE user_id = $1 AND (created_at, id) < ($2, $3)
ORDER BY created_at DESC, id DESC
LIMIT 21;
```

Index pendukung: `CREATE INDEX ON api_keys (user_id, created_at DESC, id DESC);`

## Ledger saldo

```sql
CREATE TABLE accounts (
  id       bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  user_id  bigint NOT NULL UNIQUE REFERENCES users (id),
  currency text   NOT NULL,
  balance  bigint NOT NULL DEFAULT 0 CHECK (balance >= 0)
);

CREATE TABLE ledger_entries (
  id               bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  account_id       bigint NOT NULL REFERENCES accounts (id),
  amount           bigint NOT NULL CHECK (amount <> 0),  -- positif = masuk, negatif = keluar
  kind             text   NOT NULL CHECK (kind IN ('topup', 'usage', 'refund', 'adjustment')),
  idempotency_key  text   NOT NULL UNIQUE,
  created_at       timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX ledger_entries_account_id_idx ON ledger_entries (account_id, created_at DESC);

-- aplikasi tidak boleh mengubah/menghapus catatan
REVOKE UPDATE, DELETE ON ledger_entries FROM app_user;
```

Satu mutasi:

```sql
BEGIN;
INSERT INTO ledger_entries (account_id, amount, kind, idempotency_key)
VALUES ($1, $2, $3, $4);                          -- gagal (23505) kalau kunci sudah pernah dipakai
UPDATE accounts SET balance = balance + $2 WHERE id = $1;  -- CHECK menolak kalau jadi negatif
COMMIT;
```

Cek kecocokan berkala: `SUM(amount)` per akun di ledger harus sama dengan `balance`.

## Masa simpan

```sql
-- dijalankan terjadwal; ulangi sampai 0 baris terhapus
DELETE FROM request_logs
WHERE id IN (
  SELECT id FROM request_logs
  WHERE created_at < now() - interval '90 days'
  LIMIT 5000
);
```

Tabel yang sangat besar dan berbasis waktu: partisi per bulan (`PARTITION BY RANGE (created_at)`), lalu
buang partisi lama dengan `DROP TABLE` (instan).

## User aplikasi dan batas waktu

```sql
CREATE ROLE app_user LOGIN PASSWORD '…';           -- bukan superuser; sandi dari secret, bukan dari repo
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_user;
ALTER ROLE app_user SET statement_timeout = '10s';
```

## SQL dinamis di dalam fungsi database

Kalau fungsi PL/pgSQL harus menyusun SQL sendiri, nama dimasukkan lewat `format('%I')` dan nilai lewat
`USING` (atau `%L`), tidak pernah dengan `||`:

```sql
EXECUTE format('SELECT count(*) FROM %I WHERE status = $1', tbl)
  INTO n
  USING wanted_status;
```

## Pencarian LIKE yang aman

```sql
-- $1 sudah di-escape aplikasi: \ → \\, % → \%, _ → \_
SELECT id, name FROM products
WHERE name ILIKE '%' || $1 || '%' ESCAPE '\'
LIMIT 50;
```

## Mencari query terberat (pg_stat_statements)

Aktifkan sekali (butuh restart database):

```yaml
# compose
db:
  image: postgres:16-alpine
  command: ["postgres", "-c", "shared_preload_libraries=pg_stat_statements"]
```

```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- 10 query yang paling banyak memakan waktu database
SELECT calls,
       round(total_exec_time::numeric, 1) AS total_ms,
       round(mean_exec_time::numeric, 2)  AS rata_ms,
       left(query, 80)                    AS query
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

Tanda N+1: query sederhana (`SELECT … WHERE id = $1`) dengan `calls` jauh lebih besar daripada jumlah request
endpoint-nya. Setelah perbaikan, `SELECT pg_stat_statements_reset();` lalu ukur ulang.

## Membaca rencana query

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT …;
```

Tanda bahaya: `Seq Scan` pada tabel besar untuk query yang sering dipanggil, estimasi baris yang meleset jauh.

## Sumber

- [PostgreSQL: ALTER TABLE](https://www.postgresql.org/docs/current/sql-altertable.html): `NOT VALID`, `SET NOT NULL`, default kolom.
- [PostgreSQL: CREATE INDEX (CONCURRENTLY)](https://www.postgresql.org/docs/current/sql-createindex.html).
- [PostgreSQL: PL/pgSQL dynamic commands (format, USING)](https://www.postgresql.org/docs/current/plpgsql-statements.html#PLPGSQL-STATEMENTS-EXECUTING-DYN)
- [PostgreSQL: pg_stat_statements](https://www.postgresql.org/docs/current/pgstatstatements.html)
- [PostgreSQL: Pattern matching (LIKE, ESCAPE)](https://www.postgresql.org/docs/current/functions-matching.html)
- [PostgreSQL: Table Partitioning](https://www.postgresql.org/docs/current/ddl-partitioning.html).
- [PostgreSQL: UUID Functions](https://www.postgresql.org/docs/current/functions-uuid.html).
- [PostgreSQL: Client Connection Defaults](https://www.postgresql.org/docs/current/runtime-config-client.html): `lock_timeout`, `statement_timeout`.
- [Use The Index, Luke: No offset](https://use-the-index-luke.com/no-offset).
