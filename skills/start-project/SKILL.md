---
name: start-project
description: Skill ini memulai proyek web dari nol (wawancara kebutuhan, memilih stack, membuat struktur dan file proyek, menyambungkan frontend-API-database pertama kali) atau memasang webcraft-skills ke proyek yang sudah ada (membaca repo, menulis file proyek, mengaudit ketidaksesuaian). Pakai saat user bilang "bikin web/aplikasi baru", "mulai proyek", atau saat proyek belum punya docs/PROJECT.md.
license: MIT
metadata:
  version: "0.1"
  language: id
  status: draf
---

# Memulai proyek

## Tujuan

Proyek punya fondasi yang sama sejak hari pertama: kebutuhan jelas, stack tercatat, struktur rapi, file proyek
lengkap, dan satu alur ujung ke ujung (frontend → API → database) sudah terbukti jalan. Proyek lama yang
dipasangi webcraft-skills mendapat peta yang jujur tentang kondisinya.

## Kapan dipakai

- **Mode A, proyek baru:** belum ada kode.
- **Mode B, proyek yang sudah ada:** ada kode, tapi belum ada `docs/PROJECT.md`.

**Tidak dipakai untuk:** menambah fitur di proyek yang sudah punya file proyek (`build-feature`).

## Data proyek yang dibutuhkan

Diperoleh dari wawancara (mode A) atau dari isi repo (mode B). Tidak ada yang dikarang.

## Mode A: proyek baru

### 1. Wawancara

Tanyakan sekaligus dalam satu pesan, bahasa sederhana, **dengan rekomendasi di tiap pertanyaan** supaya user
cukup menjawab "ya":

1. Web ini untuk apa dan untuk siapa? (satu-dua kalimat)
2. 3-5 hal utama yang harus bisa dilakukan pengguna di **versi pertama**.
3. Perlu login? Dengan apa (email, Google, Telegram)?
4. Ada pembayaran atau saldo?
5. Bahasa situs (Indonesia, Inggris, keduanya)?
6. Akan dijalankan di mana (VPS sendiri, hosting/platform), dan kira-kira berapa pengguna awal?
7. Ada preferensi teknologi? Kalau tidak, pakai bawaan di `CATALOG.md`.

Versi pertama dibuat **sekecil mungkin**. Fitur lain dicatat sebagai "nanti" di daftar fitur.

### 2. Usulan

Tulis ringkasan: tujuan, fitur versi pertama, stack, tempat jalan, layanan luar yang dibutuhkan. Tunggu
persetujuan user.

### 3. Fondasi

1. `git init`. Satu repo untuk satu aplikasi. [12-Factor: Codebase]
2. Struktur folder sesuai `structure-project`.
3. Semua dependensi dideklarasikan di file manifest (`package.json`, `go.mod`) dengan lockfile di-commit.
   [12-Factor: Dependencies]
4. `.gitignore` (minimal: `.env`, folder build, `node_modules`), `.env.example`.
5. File proyek:
   - `AGENTS.md` dari `assets/AGENTS.md`
   - `docs/PROJECT.md` dari `assets/PROJECT.md`
   - `docs/features/README.md` berisi fitur versi pertama (status: rencana) dan daftar "nanti"
   - `docs/decisions/0001-stack-dan-arsitektur.md` (format ADR di `build-feature`)
6. README sesuai `write-docs` (minimal: apa ini, cara menjalankan).

### 4. Lingkungan lokal

- Database dan layanan pendukung (PostgreSQL, Redis bila perlu) dijalankan dengan Docker Compose, versi sama
  dengan produksi. Lokal dan produksi dibuat semirip mungkin. [12-Factor: Dev/prod parity]
- Satu perintah untuk menjalankan semuanya, ditulis di README.

### 5. Alur ujung ke ujung pertama

Sebelum fitur apa pun, buktikan semua bagian tersambung:

- backend punya endpoint health yang mengecek koneksi database (`build-backend`, `design-api`);
- frontend punya satu halaman yang memanggil endpoint itu dan menampilkan hasilnya;
- satu test otomatis untuk endpoint itu (`write-tests`).

### 6. Desain dan CI

- `setup-design-system` sebelum halaman pertama dibuat.
- CI minimal (typecheck, lint, test) lewat `deploy-app`.

### 7. Commit pertama

Lalu laporkan ke user: apa yang sudah ada, cara menjalankan, fitur pertama yang disarankan dikerjakan
(lewat `build-feature`).

## Mode B: proyek yang sudah ada

### 1. Baca dulu, jangan ubah apa pun

README, manifest dependensi, struktur folder, config Docker/CI, dokumen yang ada, riwayat git singkat.

### 2. Tulis `docs/PROJECT.md`

Dari `assets/PROJECT.md`, berisi **fakta yang ditemukan**. Yang tidak bisa dipastikan ditulis "belum
diketahui" dan ditanyakan ke user.

### 3. `AGENTS.md`

Kalau sudah ada, **pertahankan isinya** dan tambahkan bagian webcraft-skills. Kalau belum, buat dari
`assets/AGENTS.md`.

### 4. Audit

Bandingkan proyek dengan skill yang relevan dan tulis `docs/audit-webcraft.md`:

| Skill | Temuan | Bukti (file:baris) | Risiko | Prioritas |
|---|---|---|---|---|

Urutan prioritas: (1) keamanan dan rahasia, (2) data dan uang, (3) bug yang dirasakan pengguna, (4) struktur
dan kerapian.

### 5. Laporan

Ringkas ke user dalam bahasa sederhana. **Tidak ada perbaikan di langkah ini.** Setiap perbaikan dikerjakan
kemudian lewat `fix-bug` atau `build-feature`, satu per satu.

## Larangan

- Membangun semua fitur sekaligus di versi pertama.
- Menambah layanan (Redis, antrean, platform) yang belum dibutuhkan versi pertama.
- Mode B: mengubah kode, menghapus file, atau "sekalian merapikan" saat audit.
- Menimpa `AGENTS.md`/dokumen yang sudah ada tanpa izin.

## Cek hasil

Mode A:

- [ ] `AGENTS.md`, `docs/PROJECT.md`, `docs/features/README.md`, ADR 0001, README, `.env.example` ada.
- [ ] Satu perintah menjalankan proyek lokal; alur health frontend → API → database berhasil.
- [ ] Test dan typecheck lulus; CI berjalan.

Mode B:

- [ ] `docs/PROJECT.md` hanya berisi fakta; yang belum pasti ditandai.
- [ ] `docs/audit-webcraft.md` setiap temuan punya bukti file:baris.
- [ ] Tidak ada perubahan kode.

## Aset

- `assets/PROJECT.md`: kerangka ringkasan proyek.
- `assets/AGENTS.md`: kerangka aturan proyek untuk agent.

## Sumber

- [The Twelve-Factor App](https://12factor.net/): I. Codebase, II. Dependencies, X. Dev/prod parity.
- [AGENTS.md](https://agents.md/): format terbuka file instruksi agent.
