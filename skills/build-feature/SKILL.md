---
name: build-feature
description: Skill ini mengatur alur menambah atau mengubah fitur dari awal sampai selesai - rencana (spek fitur), catatan keputusan (ADR), urutan pengerjaan database ke API ke UI, test, review diri, cek nyata, commit, dan laporan. Pakai saat user minta fitur baru, halaman baru, perubahan alur, atau perubahan yang menyentuh lebih dari satu bagian web, juga saat perlu mencatat keputusan penting seperti menambah alat atau layanan baru.
license: MIT
metadata:
  version: "0.8"
  language: id
  status: draf
---

# Membangun fitur

## Tujuan

Setiap fitur dikerjakan dengan urutan yang sama: dipahami dan disetujui dulu, dikerjakan per lapisan, diuji,
dicek seperti user memakainya, lalu dicatat. Hasilnya bisa dirawat orang lain, dan fitur baru tidak merusak
fitur lama.

## Kapan dipakai

- Fitur baru, halaman baru, perubahan alur.
- Perubahan yang menyentuh lebih dari satu bagian (mis. database + UI).
- Keputusan penting perlu dicatat (lihat langkah 3).

**Tidak dipakai untuk:** memperbaiki bug atau merapikan kode tanpa mengubah perilaku (`fix-bug`), proyek baru
dari nol (`start-project`).

## Data proyek yang dibutuhkan

- `docs/PROJECT.md` (stack, struktur). Tidak ada → `start-project`.
- `docs/features/README.md` dan spek fitur terkait kalau sudah ada.
- `docs/decisions/` untuk keputusan yang sudah diambil. Jangan bertentangan dengan keputusan berstatus
  "diterima" tanpa membuat keputusan baru yang menggantikannya.

## Langkah

### 1. Pahami

- Ulangi permintaan dengan kalimat sederhana. Sebut bagian web yang tersentuh (peta di `start-here`).
- Cari kode yang sudah ada untuk hal serupa sebelum membuat yang baru (indeks kode kalau ada, `start-here`
  aturan 12), supaya tidak membuat fungsi atau komponen kembar.
- Tanyakan yang belum jelas, **satu kali, sekaligus**, dengan rekomendasi untuk tiap pertanyaan.

### 2. Rencana (spek fitur)

Ukur dulu besarnya:

| Ukuran | Ciri | Rencana |
|---|---|---|
| Kecil | satu bagian saja, tidak menyentuh data, uang, login, atau data pribadi | ringkasan 3-5 baris di chat |
| Besar | lebih dari satu bagian, **atau** menyentuh data/uang/login/data pribadi/file/pihak luar | file `docs/features/<fitur>.md` dari `assets/feature-spec.md` |

- Fitur besar yang menyentuh login, uang, data pribadi, file, atau pihak luar: jalankan `secure-feature`
  sebelum rencana disetujui. Di tahap rencana, threat model dilakukan pada **desain** (alur, data, batas
  kepercayaan, keputusan pencegahan + test yang wajib); pemeriksaan kode dilakukan lagi setelah coding.
- Tunjukkan rencana ke user dalam bahasa sederhana. **Jangan mulai coding sebelum disetujui.**
- Daftarkan fitur di `docs/features/README.md` (nama, satu baris kegunaan, status).

### 3. Catat keputusan penting (ADR)

Buat `docs/decisions/NNNN-judul.md` dari `assets/adr.md` kalau rencana:

- menambah alat, library besar, atau layanan baru (lihat `CATALOG.md`);
- mengubah struktur, arsitektur, atau cara kerja yang sudah tercatat;
- memilih pendekatan yang mahal dibatalkan (cara login, cara bayar, bentuk data utama).

Aturan ADR [Nygard]:

- Nomor urut, tidak pernah dipakai ulang.
- Isi: konteks (fakta), keputusan ("Kita akan …"), status, konsekuensi (baik dan buruk).
- Pendek, 1-2 halaman.
- Status `diusulkan` selama rencana belum disetujui; `diterima` setelah user setuju.
- **Tidak diedit setelah diterima.** Kalau keputusan berubah, buat ADR baru; ADR lama diberi status
  "digantikan oleh NNNN".

### 4. Kerjakan dari dalam ke luar

Urutan, dan skill tiap lapisan:

1. Database: `change-database`
2. Logika backend: `build-backend`
3. API: `design-api`
4. UI: `build-ui` (+ `setup-design-system` kalau butuh token baru)
5. Teks: `manage-translations`
6. Notifikasi, pembayaran, auth: skill fiturnya masing-masing

Lewati lapisan yang tidak tersentuh. Letak setiap file mengikuti `structure-project`.

Selama mengerjakan:

- **Tetap di lingkup spek.** Masalah lain yang ditemukan dicatat di laporan, tidak ikut dikerjakan.
- **Langkah kecil.** Setiap langkah membuat proyek tetap bisa jalan (typecheck/test lulus), bukan satu
  perubahan raksasa di akhir. [Google: Small CLs]

### 5. Test

Ikuti `write-tests`. Minimal: alur utama berhasil + satu alur gagal yang penting (input salah, tidak punya
izin, pihak luar gagal).

### 6. Review diri

Sebelum menyebut selesai, cek diff sendiri [Google: What to look for in a code review]:

- [ ] Sesuai spek; tidak ada yang di luar lingkup.
- [ ] Letak file sesuai `structure-project`; tidak ada duplikat.
- [ ] UI punya keadaan memuat, kosong, dan error.
- [ ] Teks yang tampil ke user lewat kamus terjemahan.
- [ ] Tidak ada rahasia, `console.log`/print debug, kode mati, atau TODO tanpa catatan.
- [ ] Error ditangani; pesan ke user tidak membocorkan detail internal.
- [ ] Dokumentasi yang terpengaruh ikut diperbarui (`write-docs`).

### 7. Cek nyata

Jalankan aplikasi dan coba alurnya **seperti user**: di browser, ukuran HP dan desktop, alur berhasil dan
gagal. Test lulus belum berarti fitur jalan.

### 8. Commit

- Ikuti konvensi commit proyek kalau sudah ada. Kalau belum: Conventional Commits
  (`feat: …`, `fix: …`, `docs: …`, `refactor: …`, `test: …`; perubahan yang merusak kompatibilitas diberi
  `!`). [Conventional Commits]
- Satu commit satu tujuan. Pesan menjelaskan **kenapa**, bukan hanya apa.
- **Push, merge, dan deploy hanya dengan izin user.**

### 9. Laporan ke user

Bahasa sederhana, berisi:

- apa yang berubah (yang terlihat user);
- apa yang **sudah dicek** dan hasilnya;
- apa yang **belum dicek** atau masih kurang;
- hal lain yang ditemukan di luar lingkup;
- langkah yang perlu dilakukan user (kalau ada).

Perbarui status fitur di `docs/features/README.md`.

## Larangan

- Mulai coding fitur besar tanpa rencana yang disetujui.
- Menambah alat/layanan tanpa ADR.
- Mengedit ADR yang sudah diterima.
- "Sekalian" merapikan atau mengubah hal di luar spek.
- Menyebut selesai tanpa test dan cek nyata.

## Cek hasil

- [ ] Rencana disetujui user sebelum coding (fitur besar: file spek ada).
- [ ] Keputusan penting punya ADR.
- [ ] Test lulus; hasil perintahnya dilampirkan di laporan.
- [ ] Test yang butuh database/Redis sudah dijalankan di database test sekali pakai. Kalau belum, tugas
  dilaporkan "belum selesai", bukan "selesai".
- [ ] Cek nyata dilakukan, atau alasan tidak bisa disebut di laporan.
- [ ] `docs/features/README.md` diperbarui.

## Aset

- `assets/feature-spec.md`: kerangka spek fitur.
- `assets/adr.md`: kerangka catatan keputusan.

## Sumber

- [Michael Nygard: Documenting Architecture Decisions](https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions): format dan aturan ADR.
- [Conventional Commits 1.0.0](https://www.conventionalcommits.org/en/v1.0.0/).
- [Google Engineering Practices: Small CLs](https://google.github.io/eng-practices/review/developer/small-cls.html).
- [Google Engineering Practices: What to look for in a code review](https://google.github.io/eng-practices/review/reviewer/looking-for.html).
