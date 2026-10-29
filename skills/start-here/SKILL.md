---
name: start-here
description: Skill ini adalah pintu masuk webcraft-skills. Isinya peta bagian web (frontend, API, backend, database, infra), letak file proyek, aturan kerja umum, dan cara memilih skill yang tepat. Pakai saat permintaan user masih umum atau berbahasa awam ("bikin web", "tambah login", "webnya error", "mau deploy"), saat belum jelas skill mana yang cocok, atau saat mulai bekerja di proyek yang memakai webcraft-skills.
license: MIT
metadata:
  version: "0.4"
  language: id
  status: draf
---

# Mulai di sini

## Tujuan

Permintaan user, termasuk dari orang yang tidak paham teknis, diterjemahkan menjadi langkah kerja yang
tepat dengan skill yang tepat. Siapa pun agent-nya, cara kerja dan hasilnya sama.

## Kapan dipakai

- Awal sesi di proyek yang memakai webcraft-skills.
- Permintaan masih umum atau berbahasa awam.
- Ragu skill mana yang berlaku.

**Tidak dipakai untuk:** mengerjakan tugasnya sendiri. Skill ini hanya mengarahkan; pengerjaan ada di skill
tujuan.

## Peta web

Bayangkan web seperti restoran:

| Bagian | Analogi | Tugasnya | Skill |
|---|---|---|---|
| **Frontend** (UI) | ruang makan | yang dilihat dan diklik user di browser/HP | `setup-design-system`, `build-ui`, `manage-translations` |
| **API** | pelayan + menu | pintu resmi antara frontend dan backend; aturan apa yang boleh diminta | `design-api` |
| **Backend** | dapur | logika: cek izin, hitung harga, proses pesanan | `build-backend` |
| **Database** | gudang | tempat data disimpan permanen | `change-database` |
| **Cache/antrean** (mis. Redis) | meja saji | data sementara supaya cepat, pekerjaan yang menunggu giliran | `build-backend` |
| **Pihak luar** | pemasok | pembayaran, email, Telegram, login Google | `implement-payment`, `send-notifications`, `implement-auth` |
| **Infra** | gedung + listrik | server, domain, HTTPS, deploy otomatis, backup, pemantauan | `deploy-app`, `operate-app` |

Alur satu permintaan: browser → frontend → API → backend → database (dan pihak luar bila perlu) → kembali ke
browser. Aturan dasarnya: **frontend tidak pernah langsung ke database**, dan **rahasia (kunci, sandi) hanya
ada di backend/server**.

## File proyek

Data khas proyek tidak ada di skill, tapi di repo proyek. Semua skill membaca dari sini:

| File | Isi | Dibuat oleh |
|---|---|---|
| `AGENTS.md` (root) | aturan singkat proyek + "proyek ini memakai webcraft-skills" | `start-project` |
| `docs/PROJECT.md` | tujuan, pengguna, stack, cara menjalankan, layanan luar | `start-project` |
| `docs/DESIGN.md` | token warna/font/spasi, daftar komponen UI | `setup-design-system` |
| `docs/features/README.md` | daftar fitur | `build-feature` |
| `docs/features/<fitur>.md` | spek satu fitur | `build-feature` |
| `docs/decisions/NNNN-judul.md` | catatan keputusan penting (ADR) | `build-feature` |

Kalau `docs/PROJECT.md` belum ada, jalankan `start-project` dulu: mode A kalau repo belum berisi kode, mode B
("proyek yang sudah ada") kalau sudah. `AGENTS.md` di subfolder (mis. buatan framework) tidak menggantikan
`AGENTS.md` root.

## Memilih skill

| User bilang kira-kira… | Skill |
|---|---|
| "bikin web/aplikasi baru", "mulai proyek" | `start-project` |
| "tambah fitur …", "bikin halaman …", "ubah alur …" | `build-feature` (yang memanggil skill per bagian) |
| "error", "tidak jalan", "salah hitung", "rapikan kode ini" | `fix-bug` |
| "file ini taruh di mana", "foldernya berantakan", "tambah env" | `structure-project` |
| "login", "daftar", "lupa password", "2FA", "peran admin" | `implement-auth` |
| "top-up", "bayar", "saldo", "webhook pembayaran" | `implement-payment` |
| "kirim email/notif/Telegram" | `send-notifications` |
| "panel admin" | `build-admin-panel` |
| "warna/tema/font web" | `setup-design-system` |
| "tambah bahasa", "terjemahan" | `manage-translations` |
| "aman nggak?", "pakai library X", "tambah fitur AI/chatbot", "kena bot/spam" | `secure-feature` (+ `build-feature` untuk membangunnya) |
| "kunci API untuk user" | `implement-auth` |
| "tambah test" | `write-tests` |
| "README", "dokumentasi API" | `write-docs` |
| "deploy", "domain", "HTTPS", "CI" | `deploy-app` |
| "backup", "server down", "pantau" | `operate-app` |

Satu permintaan bisa menyentuh beberapa skill. Pakai `build-feature` sebagai pengatur urutannya.

## Aturan kerja (berlaku untuk semua skill)

1. **Tanya kalau data kurang.** Jangan mengarang stack, nama, nilai, atau keputusan.
2. **Satu rekomendasi.** Kalau ada pilihan teknis, beri rekomendasi + alasan dalam bahasa sederhana. Jangan
   menyerahkan daftar pilihan mentah ke user awam.
3. **Tetap di lingkup.** Kerjakan yang diminta saja. Hal lain yang ditemukan dicatat dan dilaporkan, tidak
   langsung dikerjakan.
4. **Tidak menambah alat/layanan tanpa keputusan tercatat** (lihat `CATALOG.md` dan ADR di `build-feature`).
5. **Rahasia tidak pernah** masuk kode, repo, log, atau chat.
6. **Minta izin dulu** untuk aksi yang sulit dibatalkan atau keluar ke publik: menghapus data, migrasi di
   produksi, deploy, push, mengirim pesan, transaksi uang.
7. **Jujur saat melapor.** Pisahkan "sudah dicek dan berhasil" dari "belum dicek". Jangan bilang selesai kalau
   test atau pengecekan belum dijalankan.
8. **Bahasa sederhana ke user.** Istilah teknis dijelaskan sekali, singkat.
9. **Wajib vs contoh.** "Aturan wajib" dan "Larangan" di setiap skill berlaku di semua proyek. Baris
   "Contoh bentuk:" dan isi `references/` boleh diganti bentuk lain yang tercatat di bagian *Konvensi*
   `docs/PROJECT.md`. Bentuk proyek yang berbeda dari contoh **bukan** pelanggaran selama aturannya terpenuhi.
10. **Kalau sumber bertentangan**, percayai dengan urutan ini:
    1. kode yang dipakai: bisa dicapai dari entry point (`main`, pendaftaran route, file halaman) dan
       bagiannya ikut dibangun/dijalankan menurut config deploy. Kode yang tidak dipanggil dari mana pun adalah
       kode mati;
    2. config produksi di repo (compose produksi, config reverse proxy, workflow CI/deploy);
    3. dokumen keputusan (ADR, spek yang mencatat keputusan) — menjelaskan **maksud**, bukan menggantikan kode;
    4. dokumen deploy/operasional terbaru;
    5. README dan dokumen lain.

    Sumber yang kalah dicatat sebagai temuan `write-docs`. Kalau kode dan keputusan berbeda, itu temuan
    (kode salah atau keputusan usang); tanyakan ke user mana yang dimaksud.
11. **Kode baru ikut skill, kode lama tidak dibongkar diam-diam.** Bagian proyek yang tercatat sebagai *utang*
    dirapikan saat bagian itu disentuh atau lewat tugas tersendiri, bukan "sekalian" di tugas lain.
    Pengecualian: **letak folder**. Kode baru mengikuti struktur yang ada sampai pemindahan diputuskan, supaya
    proyek tidak punya dua pola sekaligus.
12. **Tanya atau tulis "belum diketahui"?** Saat bekerja bersama user: tanya. Saat audit atau tugas tanpa user
    (mode B): tulis "belum diketahui" dan kumpulkan pertanyaannya di laporan akhir.

## Langkah

1. Cek `AGENTS.md` (root) dan `docs/PROJECT.md`.
   - Ada: baca keduanya, termasuk bagian *Konvensi* dan *Dikelola di luar repo*.
   - Tidak ada: jangan mulai skill lain dulu. Jalankan `start-project` (mode A untuk repo kosong, mode B untuk
     repo yang sudah berisi kode). Pengecualian: user hanya bertanya, tidak meminta perubahan.
2. Ulangi permintaan user dengan kalimat sederhana, sebut bagian web yang tersentuh (pakai peta di atas).
3. Pilih skill dari tabel. Kalau tidak ada yang cocok, katakan terus terang dan kerjakan dengan aturan kerja di
   atas.
4. Ikuti skill tujuan sampai bagian "Cek hasil"-nya terpenuhi.

## Cek hasil

- [ ] Skill yang dipakai disebutkan ke user.
- [ ] Data yang kurang ditanyakan, tidak dikarang.
- [ ] Laporan akhir memisahkan yang sudah dicek dan yang belum.

## Sumber

- [MDN: How the web works](https://developer.mozilla.org/en-US/docs/Learn_web_development/Getting_started/Web_standards/How_the_web_works): alur browser–server.
- [The Twelve-Factor App: Config](https://12factor.net/config): rahasia di luar kode.
- [Agent Skills specification](https://agentskills.io/specification): format skill.
