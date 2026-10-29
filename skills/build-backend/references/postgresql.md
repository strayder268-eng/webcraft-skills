# Antrean dan outbox di PostgreSQL

Cukup untuk sebagian besar aplikasi kecil-menengah tanpa menambah layanan baru.

## Tabel

```sql
CREATE TABLE jobs (
  id           bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  kind         text   NOT NULL,                 -- 'send_email', 'process_topup', …
  payload      jsonb  NOT NULL,
  status       text   NOT NULL DEFAULT 'pending'
               CHECK (status IN ('pending', 'running', 'done', 'failed')),
  attempts     int    NOT NULL DEFAULT 0,
  max_attempts int    NOT NULL DEFAULT 5,
  run_after    timestamptz NOT NULL DEFAULT now(),
  last_error   text,
  created_at   timestamptz NOT NULL DEFAULT now(),
  updated_at   timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX jobs_ready_idx ON jobs (run_after) WHERE status = 'pending';
```

## Outbox: simpan pekerjaan bersama perubahan data

```sql
BEGIN;
UPDATE orders SET status = 'paid' WHERE id = $1;
INSERT INTO jobs (kind, payload) VALUES ('send_receipt', jsonb_build_object('order_id', $1));
COMMIT;   -- keduanya tersimpan, atau tidak sama sekali
```

## Worker mengambil pekerjaan

```sql
UPDATE jobs SET status = 'running', attempts = attempts + 1, updated_at = now()
WHERE id = (
  SELECT id FROM jobs
  WHERE status = 'pending' AND run_after <= now()
  ORDER BY run_after
  LIMIT 1
  FOR UPDATE SKIP LOCKED      -- beberapa worker tidak mengambil pekerjaan yang sama
)
RETURNING id, kind, payload, attempts, max_attempts;
```

- Berhasil → `status = 'done'`.
- Gagal dan `attempts < max_attempts` → `status = 'pending'`, `run_after = now() + jeda` (makin lama tiap
  percobaan), simpan `last_error`.
- Gagal dan sudah habis percobaan → `status = 'failed'` untuk diperiksa manusia.
- Pekerjaan `running` yang macet terlalu lama (worker mati) dikembalikan ke `pending` oleh pembersih berkala.
- Baris `done` lama dihapus terjadwal (masa simpan, `change-database`).

## Sumber

- [PostgreSQL: SELECT … FOR UPDATE SKIP LOCKED](https://www.postgresql.org/docs/current/sql-select.html#SQL-FOR-UPDATE-SHARE)
- [microservices.io: Transactional outbox](https://microservices.io/patterns/data/transactional-outbox.html)
