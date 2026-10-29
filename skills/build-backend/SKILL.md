---
name: build-backend
description: Skill ini menulis logika server yang rapi dan tahan gangguan - pembagian lapisan, penanganan error, log terstruktur, batas waktu dan retry ke layanan luar, cache, pekerjaan latar/antrean, rate limit, penyimpanan kunci idempotency, health check, dan mati dengan rapi. Pakai saat menulis atau mengubah logika backend, memanggil layanan luar, menambah cache atau Redis, membuat pekerjaan terjadwal/latar, menambah rate limit, atau menambah log.
license: MIT
metadata:
  version: "0.1"
  language: id
  status: draf
---

# Membangun backend

## Tujuan

Logika server mudah dibaca, error tidak hilang dan tidak bocor ke user, gangguan layanan luar tidak
menjatuhkan aplikasi, dan resource server (memori, koneksi, CPU) tidak terbuang.

## Kapan dipakai

- Menulis/mengubah logika di balik endpoint.
- Memanggil layanan luar (pembayaran, email, AI, API lain).
- Menambah cache, antrean, pekerjaan terjadwal, rate limit, log.

**Tidak dipakai untuk:** bentuk endpoint (`design-api`), tabel dan query (`change-database`), sesi dan izin
(`implement-auth`).

## Data proyek yang dibutuhkan

- Stack backend dan apakah proyek memakai Redis (`docs/PROJECT.md`). Menambah Redis atau antrean = ADR, dan
  hanya kalau syarat di `CATALOG.md` terpenuhi.

## Aturan wajib

### 1. Lapisan

`handler` (HTTP masuk/keluar) → `service` (aturan bisnis) → `store` (database) / `client` (layanan luar).

- Aturan bisnis hanya di `service`. Handler tidak berisi aturan; store tidak tahu soal HTTP.
- Service bisa diuji tanpa HTTP dan tanpa layanan luar sungguhan (dependensi lewat interface/parameter).

### 2. Error

- Error **tidak pernah diabaikan**. Setiap error ditangani atau diteruskan ke atas dengan konteks
  ("menyimpan api key: …").
- Error yang dikenal (saldo kurang, tidak ditemukan, duplikat) dibuat sebagai error bernama di service, lalu
  handler memetakannya ke status + `code` API (`design-api`).
- Error tak terduga: catat lengkap di log, kirim 500 generik ke user.
- Panic/crash di satu request tidak boleh mematikan server (middleware recover).

### 3. Log [OWASP Logging, 12-Factor: Logs]

- Log terstruktur (JSON) ke **stdout**; penyimpanannya urusan infra (`operate-app`).
- Setiap baris log request membawa **ID request** yang sama dengan yang tercatat di error.
- Level: `error` (perlu ditindaklanjuti), `warn` (aneh tapi tertangani), `info` (peristiwa penting),
  `debug` (mati di produksi).
- Yang **wajib** dicatat: login berhasil/gagal, perubahan izin/peran, transaksi uang, aksi admin, error.
- **Tidak pernah** dicatat: sandi, token, kunci API, isi cookie sesi, nomor kartu, data pribadi lengkap.
  Samarkan kalau perlu (`sk-…a1b2`).

### 4. Memanggil layanan luar [AWS: Timeouts, retries and backoff]

- **Selalu ada batas waktu.** Tidak ada panggilan jaringan tanpa timeout.
- Retry hanya untuk kegagalan sementara (timeout, 429, 502/503) **dan** hanya untuk operasi yang aman
  diulang (baca, atau tulis dengan kunci idempotency).
- Retry memakai jeda yang makin panjang dan acak (*exponential backoff + jitter*) dengan batas jumlah.
- Tidak memanggil layanan luar di dalam transaksi database.
- Layanan luar mati → aplikasi tetap jalan dengan pesan yang jelas (503), tidak macet menunggu.

### 5. Cache

Pakai cache hanya kalau data **sering dibaca, jarang berubah, dan mahal diambil**. Kalau tidak, jangan.

- Pola *cache-aside*: baca cache → kosong → ambil dari sumber → simpan dengan masa berlaku.
- **Setiap entri punya masa berlaku (TTL).** Tidak ada cache abadi.
- Data berubah → hapus entri cache-nya di kode yang mengubah data.
- Nama kunci tersusun dan bisa di-versi: `app:v1:user:123:profile`.
- Data milik user tertentu: ID user **wajib** ada di kunci. Jangan sampai user A melihat cache milik user B.
- Database tetap sumber kebenaran. Aplikasi harus tetap benar (meski lebih lambat) kalau cache kosong atau
  mati.

### 6. Pekerjaan latar dan antrean

Pakai kalau pekerjaan lama (kirim email, proses file, panggil API lambat) tidak boleh menahan response.

- Request menyimpan pekerjaan → response cepat → worker mengerjakan.
- Pekerjaan yang lahir dari perubahan data disimpan di **transaksi yang sama** dengan perubahan datanya
  (*transactional outbox*), supaya tidak ada data tersimpan tanpa pekerjaannya atau sebaliknya.
  [microservices.io: Transactional outbox]
- Pekerjaan bisa terkirim lebih dari sekali → **worker harus idempoten** (aman dijalankan dua kali).
- Ada batas percobaan; yang terus gagal dipindah ke daftar gagal untuk diperiksa, bukan diulang selamanya.
- Aplikasi kecil cukup antrean di PostgreSQL (`references/postgresql.md`); Redis/broker khusus hanya dengan
  ADR.

### 7. Rate limit

- Wajib di: login, daftar, reset sandi, kirim kode/OTP, pembayaran, API publik (`design-api`).
- Kunci batas: per user kalau sudah login, per IP kalau belum; endpoint login juga per akun yang dicoba.
- IP diambil dari header proxy **hanya** kalau request datang dari proxy yang dipercaya.
- Tentukan dan catat: kalau penyimpanan rate limit mati, endpoint sensitif (login, bayar) **menolak**
  (gagal tertutup); endpoint biasa boleh tetap jalan.

### 8. Kunci idempotency

Untuk endpoint di `design-api` aturan 10: simpan `(kunci, user, hash payload, status, response)` dengan masa
berlaku. Penanda "sedang diproses" diambil secara atomik (constraint `UNIQUE` di database atau `SET NX` di
Redis) supaya dua request bersamaan tidak sama-sama lolos.

### 9. Health check

- `GET /healthz` (*liveness*): proses hidup. Ringan, tanpa cek database.
- `GET /readyz` (*readiness*): siap melayani; cek koneksi database/Redis dengan timeout pendek.
- Tidak membocorkan versi, config, atau detail error.

### 10. Mati dengan rapi

Saat menerima sinyal berhenti (deploy, restart): berhenti menerima request baru, selesaikan yang sedang
berjalan dalam batas waktu, tutup koneksi database/Redis, lalu keluar.

### 11. Resource

- Satu pool koneksi database dan satu klien HTTP per layanan luar, dipakai ulang; tidak membuat koneksi baru
  per request.
- Semua kerja per request mengikuti pembatalan request (context); request yang ditinggal user tidak terus
  memakan CPU/koneksi.
- Ukuran input, hasil query, dan file dibatasi.

## Larangan

- Mengabaikan error atau menelan error tanpa log.
- Memanggil jaringan tanpa timeout; retry tanpa batas.
- Retry operasi tulis yang tidak idempoten.
- Cache tanpa TTL; cache data user tanpa ID user di kunci.
- Mencatat rahasia atau data pribadi di log.
- Goroutine/promise/timer yang jalan tanpa pernah dihentikan.

## Cek hasil

- [ ] Test service mencakup jalur error penting (layanan luar gagal/timeout).
- [ ] Setiap panggilan luar punya timeout.
- [ ] Log request punya ID request; tidak ada rahasia di log (cek output test/log lokal).
- [ ] `/healthz` dan `/readyz` merespons; aplikasi mati rapi saat dihentikan.

## Stack

- `references/go.md`: lapisan, error, slog, klien HTTP, shutdown
- `references/redis.md`: cache, rate limit, kunci idempotency
- `references/postgresql.md`: antrean dan outbox di PostgreSQL

## Sumber

- [OWASP Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html).
- [The Twelve-Factor App: Logs](https://12factor.net/logs).
- [AWS Builders' Library: Timeouts, retries and backoff with jitter](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/).
- [microservices.io: Transactional outbox](https://microservices.io/patterns/data/transactional-outbox.html).
- [Kubernetes: Liveness, Readiness, and Startup Probes](https://kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/): pembedaan liveness dan readiness.
