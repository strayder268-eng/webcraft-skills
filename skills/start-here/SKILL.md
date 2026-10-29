---
name: start-here
description: Skill ini adalah pintu masuk webcraft-skills. Isinya peta bagian web (frontend, API, backend, database, infra), letak file proyek, aturan kerja umum, dan cara memilih skill yang tepat. Pakai saat permintaan user masih umum atau berbahasa awam ("bikin web", "tambah login", "webnya error", "mau deploy"), saat belum jelas skill mana yang cocok, atau saat mulai bekerja di proyek yang memakai webcraft-skills.
license: MIT
metadata:
  version: "0.1"
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

Kalau `docs/PROJECT.md` belum ada, jalankan `start-project` dulu. Untuk proyek lama, `start-project` memakai
mode "proyek yang sudah ada".

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
| "aman nggak?", "pakai library X" | `secure-feature` |
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

## Langkah

1. Baca `AGENTS.md` dan `docs/PROJECT.md` proyek. Kalau tidak ada: tawarkan `start-project`.
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
