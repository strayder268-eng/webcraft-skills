# Katalog alat dan stack

Daftar alat yang direstui, statusnya, dan kapan perlu dipakai. Agent membaca file ini sebelum memilih atau
menambah alat.

**Aturan:** menambah alat atau layanan baru ke proyek wajib dicatat alasannya sebagai keputusan (ADR, lihat
skill `build-feature`). Jangan menambah alat hanya karena "biasanya dipakai".

Status:

- `tahap-1` = file `references/` sedang/akan ditulis dan diuji pada proyek nyata.
- `belum` = belum ada file `references/`. Kalau proyek memakainya, beri tahu user dan jangan mengarang cara pakainya.

## Bawaan untuk proyek baru

Kalau user tidak punya preferensi, rekomendasikan ini (semua `tahap-1`):

| Bagian | Bawaan |
|---|---|
| Frontend | Next.js + TypeScript |
| Backend | Go |
| Database | PostgreSQL |
| Infra | Docker Compose + GitHub Actions |

Redis **tidak** termasuk bawaan; tambah hanya kalau syaratnya terpenuhi (lihat di bawah).

## Daftar

| Jenis | Alat | Status | Kapan perlu | Kapan tidak perlu |
|---|---|---|---|---|
| Bahasa | TypeScript | tahap-1 | frontend dan backend Node | — |
| | JavaScript | belum | proyek lama yang sudah memakainya | proyek baru: pakai TypeScript |
| | Go | tahap-1 | backend API, layanan yang butuh ringan dan cepat | — |
| | Python | belum | backend yang dekat dengan data/ML | — |
| | Rust | belum | bagian yang sangat butuh performa | web biasa (terlalu mahal waktunya) |
| Framework frontend | Next.js | tahap-1 | situs + aplikasi web dengan React | — |
| Framework backend | Express / NestJS (Node) | belum | tim hanya menguasai JavaScript/TypeScript | — |
| | FastAPI / Django (Python) | belum | lihat Python | — |
| Database | PostgreSQL | tahap-1 | hampir semua aplikasi | — |
| | MySQL | belum | proyek/hosting yang sudah memakainya | proyek baru: pakai PostgreSQL |
| | Redis | tahap-1 | cache, rate limit, antrean, sesi berumur pendek | aplikasi kecil yang cukup dengan PostgreSQL saja |
| Platform | Supabase | belum | ingin PostgreSQL + login + penyimpanan file siap pakai tanpa mengurus server | butuh kendali penuh atas backend |
| Infra | Docker / Docker Compose | tahap-1 | menjalankan aplikasi sama persis di laptop dan server | — |
| | GitHub Actions | tahap-1 | build, test, dan image otomatis | — |
