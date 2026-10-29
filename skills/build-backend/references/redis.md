# Redis

Hanya dipakai kalau syarat di `CATALOG.md` terpenuhi (cache, rate limit, antrean, sesi berumur pendek) dan
tercatat sebagai ADR.

## Dasar

- Redis bukan database utama. Isinya harus boleh hilang tanpa merusak data; sumber kebenaran tetap
  PostgreSQL.
- Wajib sandi (`requirepass` atau ACL), tidak dibuka ke internet, hanya bisa diakses backend.
- Atur `maxmemory` dan kebijakan buang (*eviction*). Untuk cache murni: `allkeys-lru`. Kalau Redis juga
  menyimpan rate limit/kunci idempotency yang tidak boleh terbuang sembarangan: `volatile-lru` dan semua kunci
  diberi TTL. [Redis: Key eviction]
- Nama kunci: `app:v1:<jenis>:<id>[:<bagian>]`. Naikkan `v1` → `v2` saat bentuk data berubah, supaya data lama
  tidak terbaca salah.

## Cache-aside

```
GET app:v1:model-list
  ada   → pakai
  kosong → ambil dari database → SET app:v1:model-list <json> EX 300
data berubah → DEL app:v1:model-list
```

## Rate limit (jendela tetap)

```
INCR app:v1:rl:login:ip:203.0.113.5:<menit-sekarang>
EXPIRE … 60 NX            # TTL dipasang sekali, saat kunci baru dibuat (opsi NX: Redis 7+)
hasil > batas → tolak 429
```

Jalankan `INCR` dan `EXPIRE` dalam satu `MULTI`/pipeline atau skrip Lua supaya kunci tidak tertinggal tanpa
TTL. [Redis: INCR, pola rate limiter]

## Kunci idempotency / kunci kerja

```
SET app:v1:idem:<user>:<kunci> "processing" NX EX 86400
  OK   → kunci baru, proses
  nil  → sudah ada → baca isinya: "processing" (409) atau response tersimpan
selesai → SET app:v1:idem:<user>:<kunci> <response> XX EX 86400
```

`NX` membuat pengambilan kunci atomik: dari dua request bersamaan, hanya satu yang mendapat `OK`.
[Redis: SET]

## Sumber

- [Redis: SET (NX, XX, EX)](https://redis.io/docs/latest/commands/set/)
- [Redis: INCR (pola rate limiter)](https://redis.io/docs/latest/commands/incr/)
- [Redis: EXPIRE (opsi NX)](https://redis.io/docs/latest/commands/expire/)
- [Redis: Key eviction](https://redis.io/docs/latest/develop/reference/eviction/)
