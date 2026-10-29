---
name: start-project
description: Skill ini memulai proyek web dari nol (wawancara kebutuhan, memilih stack, membuat struktur dan file proyek, menyambungkan frontend-API-database pertama kali) atau memasang webcraft-skills ke proyek yang sudah ada (membaca repo, menulis file proyek, mengaudit ketidaksesuaian). Pakai saat user bilang "bikin web/aplikasi baru", "mulai proyek", atau saat proyek belum punya docs/PROJECT.md.
license: MIT
metadata:
  version: "0.6"
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

Cek dulu alat bantu agent (`start-here` aturan 12) dan catat di PROJECT.md. Lalu baca README, manifest
dependensi, struktur folder, config Docker/CI, **semua** dokumen di `docs/`, file
`AGENTS.md` di mana pun (root dan subfolder), riwayat git singkat.

### 2. Tulis `docs/PROJECT.md`

Dari `assets/PROJECT.md`, berisi **fakta yang ditemukan**. Yang tidak bisa dipastikan ditulis "belum
diketahui" dan ditanyakan ke user.

- **Sumber bertentangan** (README bilang port 8080, compose memakai 8180): tulis yang dipakai kode/config yang
  berjalan; dokumen yang salah masuk audit sebagai temuan `write-docs`.
- **Keputusan lama** yang belum berbentuk ADR (spek, catatan desain): rujuk dokumen itu di PROJECT.md. ADR
  baru hanya dibuat saat keputusan itu diubah (`build-feature` langkah 3); tidak perlu menulis ulang ADR untuk
  keputusan masa lalu. Setiap kali skill lain meminta "catat di ADR", dokumen keputusan lama yang sudah ada
  ini juga memenuhi.
- **Dikelola di luar repo**: konfigurasi yang ada di platform (hosting, CDN, DNS, penyedia email) dicatat di
  bagian itu: apa, di mana, siapa yang punya akses. Audit menandainya "cek manual oleh user", bukan menebak.

### 3. `AGENTS.md`

Kalau `AGENTS.md` root sudah ada, **pertahankan isinya** dan tambahkan bagian webcraft-skills. Kalau belum,
buat dari `assets/AGENTS.md`. `AGENTS.md` di subfolder (mis. buatan framework) dibiarkan dan dirujuk dari root.

### 4. Konvensi

Proyek lama hampir selalu punya bentuk sendiri yang berbeda dari "Contoh bentuk" skill (nama folder, format
kamus, alat migrasi, alat test, nama file dokumen). Untuk setiap perbedaan, isi tabel *Konvensi* di
PROJECT.md dengan salah satu status:

| Status | Artinya | Contoh |
|---|---|---|
| **konvensi** | bentuknya beda, tapi **aturan wajib terpenuhi**. Dipakai terus; kode baru mengikuti bentuk ini. Bukan temuan audit. | `web/app` tanpa `src/`; kamus TypeScript bertipe; `node --test` |
| **utang** | bentuknya **melanggar** aturan wajib. Masuk audit. Kode baru wajib ikut aturan; kode lama dirapikan saat disentuh atau lewat tugas sendiri. | `<button>` mentah di 35 halaman lama; satu `schema.sql` yang berisi `DROP` |
| **perlu keputusan** | melanggar aturan wajib, dan memperbaikinya mengubah alat, kontrak, atau keputusan yang sudah tercatat (butuh ADR + persetujuan user). Masuk audit. | pindah ke alat migrasi bernomor; ganti format error API |

Satu topik satu baris, cukup sedetail yang dibutuhkan agent berikutnya untuk tahu bentuk mana yang dipakai
(biasanya 10-30 baris). Baris *utang* dan *perlu keputusan* menyebut ID temuannya di audit.

### 5. Audit

Tulis `docs/audit-webcraft.md` dari `assets/audit-webcraft.md`. Supaya dua agent yang mengaudit proyek yang
sama mendapat hasil yang sama, audit dikerjakan **lengkap, bukan sampel**:

1. **Per skill, per aturan, per butir.** Semua skill di indeks README diperiksa, urut seperti indeks. Skill
   yang topiknya tidak ada di proyek (mis. tidak ada pembayaran) cukup satu baris `–` dengan alasannya. Yang
   diperiksa: setiap aturan bernomor di "Aturan wajib"; untuk skill yang isinya berupa "Langkah"
   (`setup-design-system`), langkah bernomornya. Skill alur kerja (`start-here`, `start-project`,
   `build-feature`, `fix-bug`) tidak masuk daftar periksa, karena mengatur cara kerja agent, bukan isi proyek.
   - Satu aturan berisi beberapa butir. **Setiap butir** diperiksa; satu temuan tidak menghentikan
     pemeriksaan butir lain di aturan yang sama. Satu aturan bisa punya beberapa ID temuan.
   - **Status ✓ wajib menyebut buktinya** (file:baris yang menunjukkan aturan dipenuhi). ✓ tanpa bukti tidak
     sah. Kalau buktinya tidak ditemukan, statusnya bukan ✓.
   - **Bukti semua jalur.** Butir yang berbunyi *semua/selalu/setiap/tidak pernah*, atau yang menuntut
     sesuatu **tidak ada** (tidak ada rahasia di log, tidak ada `CASCADE` di ledger), dibuktikan dengan
     **pencarian**, bukan satu contoh: tulis pola yang dicari, jumlah hasilnya, dan periksa **setiap** hasil.
     Kalau ada indeks kode (`start-here` aturan 12), bukti boleh berupa hasil alat itu, mis. daftar semua
     pemanggil fungsi kirim email. Satu jalur yang benar tidak membuktikan semua jalur benar. Contoh:
     - "pengiriman selalu lewat antrean": cari semua titik kirim (`grep -rn "Send\|send.*Email" gateway/`),
       7 hasil, 6 lewat outbox, 1 langsung di `account.go:137` → T;
     - "ledger tidak pernah dihapus": cari `ON DELETE` dan `DELETE FROM` pada tabel ledger, 2 hasil
       `ON DELETE CASCADE` → T;
     - "tidak ada data pribadi di log": cari `log.` + `email`/`ip`, 0 hasil → ✓ dengan pola dan hasilnya.
   - Aturan yang sebagian butirnya sesuai: tulis `7 T(P2-02, P2-05) · ✓ butir lain: <bukti>`.
   - Aturan tanpa status = audit belum selesai.
2. **Cek kebenaran sebelum menulis temuan.** Baca sumber sesuai urutan di `start-here` aturan 10. Hanya
   kode yang dipakai yang dinilai (lihat definisinya di aturan itu); kode mati dicatat P4. Kalau
   perilaku yang tampak salah ternyata **sengaja diputuskan** di dokumen keputusan:
   - keputusan itu memenuhi aturan wajib → bukan temuan;
   - keputusan itu melanggar aturan wajib → tetap temuan, status *perlu keputusan*, sebut dokumennya;
   - dokumen lain (README) menjanjikan hal yang bertentangan dengan keputusan → temuan `write-docs`.
3. **Prioritas hanya empat, berdasarkan akibatnya** kalau dibiarkan:

   | Prioritas | Dipakai kalau akibatnya… | Contoh |
   |---|---|---|
   | **P1** keamanan dan rahasia | orang lain bisa masuk, membaca, atau mengubah yang bukan haknya; rahasia bisa bocor; pertahanan hilang saat ada gangguan | rate limit gagal terbuka di login, image berjalan sebagai root, CI tanpa pemindai |
   | **P2** data dan uang | data bisa hilang/rusak, uang bisa salah hitung, jejak uang hilang | ledger ikut terhapus, migrasi berisi `DROP`, audit uang dihapus 14 hari |
   | **P3** bug yang dirasakan pengguna | pengguna atau developer mendapat hasil salah, lambat, atau petunjuk salah | README salah port, teks tidak diterjemahkan, dokumen usang |
   | **P4** struktur dan kerapian | kode sulit dirawat, tapi belum salah | file 1.800 baris, `utils.ts`, nama modul lama |

   **Prioritas temuan = baris "Prioritas audit" di bawah aturan yang dilanggar.** Baris itu menetapkan
   prioritas bawaan setiap aturan (kadang berbeda per butir), supaya dua agent tidak menebak sendiri. Temuan
   yang melanggar beberapa aturan memakai prioritas tertinggi di antaranya. Prioritas hanya boleh dinaikkan
   atau diturunkan dengan alasan tertulis di kolom Risiko ("naik ke P1 karena endpoint ini publik"). Patokan di
   bawah dipakai untuk aturan yang belum punya baris itu dan untuk menilai alasan naik/turun:

   - **P1** juga untuk semua yang **melemahkan pertahanan**, walau belum terbukti bisa dibobol: header
     keamanan tidak sampai ke browser, token/JWT tidak dicek algoritma atau penerbitnya, input tanpa batas
     ukuran, rate limit atau 2FA yang bisa dilewati, proses berjalan sebagai root, rahasia contoh yang lolos
     validasi, API admin proxy terbuka.
   - **P2** untuk semua yang membuat **riwayat uang atau data bisa hilang**, walau saldo hari ini benar:
     ledger/audit terhapus berantai, retensi terlalu pendek, backup tanpa alert, pengiriman penting tanpa
     antrean (bisa hilang saat proses mati).
   - **P3** untuk dokumen yang salah, **kecuali** dokumen itu menyuruh melakukan hal berbahaya (simpan token
     di `localStorage`, matikan pengecekan): itu P1.
   - Satu temuan yang masuk dua prioritas memakai yang **lebih tinggi**.

   Tidak ada skala lain (tinggi/sedang/rendah). Setiap temuan diberi ID tetap (`P1-01`).
4. **Yang masuk audit:** status *utang* dan *perlu keputusan*. Status *konvensi* tidak.
5. **Semua** file teks di `docs/` dan semua file `*.md` lain di repo (root dan subfolder, kecuali
   `node_modules`, vendor, dan folder build) diperiksa: dokumen yang
   menyebut fitur, halaman, atau layanan yang sudah tidak ada adalah temuan `write-docs`. Gambar dan file biner
   cukup dicatat di *Cakupan docs/*. Di mode B dokumen usang **hanya dicatat**, tidak diarsipkan atau diubah.
6. Hal yang hanya bisa dipastikan dengan menjalankan aplikasi (waktu respons, jumlah query per request, kontras
   warna, header di domain, hasil `npm audit`/`govulncheck`) diberi status **perlu diukur** beserta cara
   mengukurnya, bukan ditebak. Audit baca-saja dianggap **lengkap** kalau semua aturan sudah punya status,
   termasuk "perlu diukur".
7. Tambahkan bagian *Yang sudah sesuai*, supaya laporan tidak hanya daftar kesalahan.
8. **Dua putaran.** Satu kali audit oleh agent mana pun pasti melewatkan sebagian butir. Karena itu audit
   dikerjakan dua kali secara terpisah, lalu digabung:
   1. **Putaran 1:** audit lengkap (langkah 1-7) → `docs/audit-webcraft.md`, ditandai "Putaran 1".
   2. **Putaran 2:** di **sesi agent baru** yang tidak membaca hasil putaran 1. Ulangi langkah 1-7 hanya untuk
      aturan berprioritas bawaan **P1 dan P2**, dengan urutan skill **terbalik** → tulis
      `docs/audit-webcraft-putaran2.md`. Baru setelah selesai, buka hasil putaran 1.
   3. **Gabung** (oleh agent putaran 2): setiap temuan yang hanya ada di satu putaran **dicek ulang di kode**.
      Nyata → masuk audit; tidak nyata → dicoret dengan alasan. Prioritas yang berbeda diselesaikan dengan baris
      "Prioritas audit". Hasil akhirnya tetap di `docs/audit-webcraft.md`, dengan kolom *Putaran* (`1`, `2`,
      `1+2`) dan bagian *Hasil penggabungan*.
   4. Kalau sesi baru tidak bisa dibuat, putaran 2 dikerjakan di sesi yang sama dengan urutan terbalik dan
      dicatat "putaran 2 di sesi yang sama" (kurang kuat, karena agent sudah melihat hasil putaran 1).

### 6. Laporan

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

- [ ] `docs/PROJECT.md` hanya berisi fakta; yang belum pasti ditandai; tabel *Konvensi* dan *Dikelola di
  luar repo* terisi.
- [ ] `docs/audit-webcraft.md` setiap temuan punya bukti file:baris; semua file `docs/` sudah diperiksa;
  yang butuh pengukuran ditandai "perlu diukur".
- [ ] Setiap ✓ di daftar periksa punya bukti; butir *semua/tidak ada* dibuktikan dengan pencarian; tidak
  ada aturan tanpa status.
- [ ] Prioritas setiap temuan sama dengan baris "Prioritas audit", atau alasan naik/turun tertulis.
- [ ] Dua putaran sudah digabung; temuan satu putaran sudah dicek ulang di kode.
- [ ] Tidak ada perubahan kode.

## Aset

- `assets/PROJECT.md`: kerangka ringkasan proyek.
- `assets/AGENTS.md`: kerangka aturan proyek untuk agent.
- `assets/audit-webcraft.md`: kerangka laporan audit mode B (temuan ber-ID, perlu diukur, daftar periksa).

## Sumber

- [The Twelve-Factor App](https://12factor.net/): I. Codebase, II. Dependencies, X. Dev/prod parity.
- [AGENTS.md](https://agents.md/): format terbuka file instruksi agent.
