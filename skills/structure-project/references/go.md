# Struktur proyek: Go (server API)

## Struktur

```
api/
  go.mod
  cmd/
    server/
      main.go                 merakit saja: baca config, sambung DB, daftarkan route, jalankan server
  internal/
    <fitur>/                  contoh: billing, apikey, auth
      handler.go              terima HTTP, validasi input, panggil service, tulis response
      service.go              aturan bisnis fitur
      store.go                akses database fitur ini
      types.go
      *_test.go
    platform/                 kode bersama lintas fitur
      config/                 baca + validasi environment variable, sekali saat start
      db/                     koneksi pool database
      httpx/                  middleware: log request, recover, auth, rate limit
      log/
  migrations/                 file migrasi bernomor (lihat change-database)
  .env.example
```

Pemetaan ke tiga tingkat penempatan:

| Tingkat | Letak |
|---|---|
| satu file/fungsi | fungsi tidak diekspor (huruf kecil) di file itu |
| satu fitur | `internal/<fitur>/` |
| lintas fitur | `internal/platform/<nama-spesifik>/` |

- Semua paket server ada di `internal/`. Toolchain Go melarang proyek lain mengimpor `internal/`, jadi isinya
  bebas diubah. [Go: Organizing a Go module]
- Program (`package main`) dikumpulkan di `cmd/<nama>/`. [Go: Organizing a Go module]

## Arah ketergantungan

```
handler → service → store → database
```

- `handler` tidak menulis SQL. `store` tidak tahu soal HTTP.
- Fitur A butuh fitur B: panggil fungsi publik service B, atau terima interface kecil di A. Go menolak impor
  melingkar, jadi kalau A dan B saling butuh, bagian bersamanya dipindah ke paket ketiga.
- `platform/` tidak mengimpor paket fitur.

## Config

- `platform/config` membaca env sekali di `main`, memvalidasi, lalu mengoper struct config ke yang
  membutuhkan. Paket lain tidak memanggil `os.Getenv` sendiri.
- Variabel wajib kosong → `main` berhenti dengan pesan jelas.
- Tidak ada variabel global yang bisa diubah (mutable global state). Dependensi dioper lewat parameter/struct.

## Nama

- Nama paket pendek, huruf kecil, satu kata, tanpa garis bawah: `billing`, `apikey`.
- Hindari `util`, `common`, `helpers`, `misc`. [Go blog: Package names]
- Jangan ulangi nama paket di nama isi: `billing.Service`, bukan `billing.BillingService`.

## Perintah cek

```sh
gofmt -l .              # harus kosong
go vet ./...
go test ./...
```

## Sumber

- [Go: Organizing a Go module](https://go.dev/doc/modules/layout)
- [Go blog: Package names](https://go.dev/blog/package-names)
- [Effective Go: Names](https://go.dev/doc/effective_go#names)
