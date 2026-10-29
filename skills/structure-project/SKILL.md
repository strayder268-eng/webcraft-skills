---
name: structure-project
description: Skill ini menentukan letak setiap file dan folder, batas antar modul, serta cara menyimpan config dan rahasia, supaya proyek tidak semrawut saat fitur bertambah. Pakai saat membuat struktur proyek baru, menambah file/komponen/modul dan bingung menaruhnya di mana, merapikan folder yang berantakan, memecah file yang terlalu besar, atau menambah environment variable dan rahasia.
license: MIT
metadata:
  version: "0.2"
  language: id
  status: draf
---

# Struktur proyek

## Tujuan

Setiap file punya tempat yang pasti. Orang baru (atau agent lain) bisa menebak letak file dengan benar tanpa
bertanya, dan menambah fitur tidak membuat folder makin berantakan.

## Kapan dipakai

- Membuat struktur proyek baru (dipanggil dari `start-project`).
- Menambah file, komponen, atau modul.
- File terlalu besar atau folder berantakan.
- Menambah environment variable atau rahasia.

**Tidak dipakai untuk:** desain tampilan (`setup-design-system`, `build-ui`), skema tabel
(`change-database`).

## Data proyek yang dibutuhkan

- Stack proyek dari `docs/PROJECT.md`. Kalau stack tidak punya file di `references/`, beri tahu user (lihat
  `CATALOG.md`).

## Aturan wajib

### 1. Kelompokkan per fitur, bukan per jenis file

Semua bagian satu fitur (komponen, logika, pemanggil API, tipe) tinggal berdekatan di satu folder fitur.
Struktur per jenis (`components/` raksasa, `hooks/` raksasa, `services/` raksasa) membuat satu fitur tersebar
ke banyak folder dan susah dihapus atau diubah. [Next.js: Split project files by feature or route]

### 2. Tiga tingkat penempatan

Berlaku untuk komponen UI, fungsi, tipe, apa pun:

| Dipakai oleh | Letak |
|---|---|
| satu halaman/tempat saja | di sebelah pemakainya (folder halaman/modul itu) |
| beberapa tempat dalam **satu fitur** | folder fitur itu |
| **lintas fitur** | folder bersama (`ui`/`shared`/`platform`, lihat `references/`) |

- **Naik tingkat:** saat sesuatu mulai dipakai di tempat kedua, **pindahkan** ke tingkat yang sesuai. Jangan
  menyalin. [keputusan webcraft]
- Folder bersama hanya berisi hal yang benar-benar umum: tidak mengambil data sendiri, tidak berisi teks atau
  aturan bisnis fitur tertentu. [keputusan webcraft]
- **Jangan naik tingkat "jaga-jaga".** Selama baru dipakai satu tempat, tetap di sebelah pemakainya.

Contoh bentuk: nama folder per stack di `references/` (mis. `src/features/`, `src/components/ui/`). Proyek
boleh memakai nama lain (mis. `web/components/<fitur>/`, `web/components/ui/` tanpa `src/`) asal tiga tingkat
ini terpenuhi dan namanya tercatat di *Konvensi* `docs/PROJECT.md`.

### 3. Arah ketergantungan satu arah

- Fitur boleh memakai folder bersama. Folder bersama **tidak boleh** memakai fitur.
- Fitur A tidak mengimpor isi dalam fitur B. Kalau A butuh sesuatu dari B, pindahkan bagian itu ke folder
  bersama, atau panggil lewat fungsi publik B yang memang disediakan.
- Frontend tidak pernah tersambung langsung ke database. Semua lewat API.

### 4. Halaman dan entry point tipis

File halaman (route) dan `main` hanya **merakit**: memanggil komponen/fungsi dari fitur. Logika tidak ditulis
di sana. [Go: cmd/ hanya entry point; keputusan webcraft untuk halaman]

### 5. Ukuran file

Patokan: file di atas ±300 baris, atau komponen/fungsi yang mengerjakan lebih dari satu hal, dipecah.
Ini patokan, bukan hukum; yang penting satu file satu tanggung jawab. [keputusan webcraft]

### 6. Nama

- Ikuti konvensi bahasa/framework (lihat `references/`).
- Nama menjelaskan isi. **Dilarang** folder atau file keranjang sampah seperti `utils/`, `utils.ts`,
  `helpers/`, `common/`, `misc/` tanpa nama yang spesifik. Pakai nama yang jelas: `format-money.ts`, `internal/platform/db`.
  [Go blog: Package names]

### 7. Config dan rahasia

- Semua yang berbeda antar tempat jalan (alamat database, kunci API, domain) dibaca dari **environment
  variable**, tidak ditulis di kode. [12-Factor: Config]
- Tes sederhana: kalau repo dibuka ke publik hari ini, apakah ada rahasia yang bocor? Harus "tidak".
  [12-Factor: Config]
- `.env.example` di-commit: berisi **semua nama variabel** dengan nilai contoh/kosong dan komentar singkat.
  File `.env` berisi nilai asli **tidak pernah** di-commit (masuk `.gitignore`).
- Config dibaca dan **divalidasi sekali saat aplikasi start**. Variabel wajib yang kosong atau salah format
  membuat aplikasi berhenti dengan pesan jelas, bukan error aneh di tengah jalan.
- Rahasia hanya di backend/server. Variabel yang ikut ke browser (mis. `NEXT_PUBLIC_*`) dianggap **publik**.
  [OWASP Secrets Management]

### 8. Struktur repo dengan frontend dan backend

Kalau satu repo berisi beberapa bagian, pisahkan di tingkat teratas:

```
web/        frontend
api/        backend
deploy/     docker compose, config server (lihat deploy-app)
docs/       PROJECT.md, DESIGN.md, features/, decisions/
AGENTS.md
```

Nama `web/` dan `api/` boleh lain kalau proyek sudah punya konvensi, asal konsisten dan tercatat di
`docs/PROJECT.md`.

## Larangan

- Menyalin komponen/fungsi lalu mengubah sedikit. Pindahkan ke tingkat yang tepat dan beri parameter.
- Menyimpan kode mati atau kode yang dikomentari "untuk jaga-jaga". Hapus; riwayatnya ada di git.
- Folder bersarang lebih dari ±4 tingkat tanpa alasan.
- Rahasia di kode, repo, contoh config, log, atau screenshot.
- Membuat struktur baru yang berbeda dari yang sudah tercatat tanpa keputusan (ADR, lihat `build-feature`).

## Langkah

**Proyek baru**

1. Baca `references/<stack>.md` untuk setiap stack di `docs/PROJECT.md`.
2. Buat folder sesuai aturan 8 dan struktur di `references/`.
3. Buat `.env.example`, `.gitignore` (berisi `.env`), dan tempat validasi config.
4. Catat ringkasan struktur di `docs/PROJECT.md` bagian "Struktur".

**Menambah file**

1. Tentukan siapa pemakainya → pilih tingkat (aturan 2).
2. Cek arah ketergantungan (aturan 3).
3. Kalau file serupa sudah ada, pakai atau perluas. Jangan membuat duplikat.

**Merapikan struktur yang ada**

1. Petakan dulu: file mana dipakai siapa.
2. Pindahkan sedikit demi sedikit, satu fitur per langkah, dengan test/typecheck tetap lulus di tiap langkah.
3. Perpindahan besar dicatat sebagai keputusan (ADR).

## Cek hasil

- [ ] Setiap file baru ada di tingkat yang sesuai pemakainya.
- [ ] Tidak ada impor dari folder bersama ke fitur, atau antar isi dalam fitur.
- [ ] Tidak ada duplikat komponen/fungsi.
- [ ] Variabel baru tercantum di `.env.example`; tidak ada rahasia di diff (`git diff` dicek).
- [ ] Typecheck/build/lint stack lulus (perintah ada di `references/`).

## Stack

- `references/nextjs.md`: Next.js + TypeScript
- `references/go.md`: Go

## Sumber

- [Next.js: Project structure and organization](https://nextjs.org/docs/app/getting-started/project-structure): colocation, private folder, split by feature.
- [Next.js: Server and Client Components](https://nextjs.org/docs/app/getting-started/server-and-client-components): `server-only`, batas client.
- [Next.js: Environment variables](https://nextjs.org/docs/app/guides/environment-variables): `NEXT_PUBLIC_` dibekukan saat build.
- [Go: Organizing a Go module](https://go.dev/doc/modules/layout): `internal/`, `cmd/`.
- [Go blog: Package names](https://go.dev/blog/package-names): hindari `util`, `common`.
- [The Twelve-Factor App: Config](https://12factor.net/config).
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html).
