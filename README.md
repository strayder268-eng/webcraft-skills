# webcraft-skills

Satu set skill standar untuk membangun web yang rapi dari UI sampai database, supaya hasil kerja AI agent
konsisten siapa pun yang mengerjakan, di IDE apa pun.

- **Umum.** Skill tidak menyebut nama proyek, warna, atau layanan tertentu. Isian khas proyek ditaruh di repo
  proyek itu sendiri (file desain, ADR, daftar fitur).
- **Satu topik, satu skill.** Tidak ada dua skill yang membahas hal yang sama.
- **Bersumber.** Setiap aturan mencantumkan sumbernya (dokumentasi resmi, OWASP, RFC, dll.).
- **Stack di dalam skill.** Aturan umum ada di `SKILL.md`; cara di framework/alat tertentu ada di
  `references/<stack>.md` di skill yang sama. Menambah stack = menambah satu file, bukan menyalin skill.

Format mengikuti spesifikasi terbuka [Agent Skills](https://agentskills.io/specification).

## Cara pakai

**Claude Code**

```sh
claude plugin marketplace add strayder268-eng/webcraft-skills
claude plugin install webcraft-skills@webcraft-skills
```

**Agent lain** (ZCode, Cursor, Codex, dll.): salin folder `skills/` ke lokasi skill yang didukung agent itu,
atau suruh agent membaca `README.md` ini dulu lalu mengikuti skill yang cocok.

## Aturan masuk (untuk agent)

1. Sebelum mengerjakan tugas, baca indeks di bawah. Kalau ada skill yang cocok, **wajib** ikuti.
2. Mulai dari `start-here` kalau permintaan user masih umum atau bahasa awam.
3. Kalau skill butuh data proyek yang belum ada (token desain, stack, keputusan), **tanya user**. Jangan mengarang.
4. Kalau stack proyek belum ada di [CATALOG.md](CATALOG.md), beri tahu user. Jangan mengarang cara pakainya.
5. Ada pilihan teknis: beri satu rekomendasi beserta alasannya, jangan lempar pilihan mentah ke user.
6. **Wajib vs contoh.** Isi "Aturan wajib" dan "Larangan" berlaku di semua proyek. Baris **"Contoh bentuk:"**
   dan seluruh isi `references/` adalah satu cara yang sudah diuji: proyek boleh memakai bentuk lain (nama
   folder, format file, alat) asal aturannya terpenuhi dan bentuk itu tercatat di bagian *Konvensi* pada
   `docs/PROJECT.md`.

## Indeks skill

Status: `belum` = belum ditulis, `draf` = ditulis tapi belum diuji, `stabil` = lolos uji.

| Kelompok | Skill | Pakai saat | Status |
|---|---|---|---|
| Alur kerja | `start-here` | permintaan masih umum; peta bagian web dan skill mana yang mengurusnya | draf |
| | `start-project` | memulai proyek baru: kebutuhan, stack, struktur, file proyek | draf |
| | `build-feature` | menambah fitur: rencana → kerjakan → review → commit → catat keputusan | draf |
| | `fix-bug` | ada yang salah atau kode perlu dirapikan tanpa mengubah perilaku | draf |
| Struktur & data | `structure-project` | menentukan letak file, batas modul, config dan rahasia | draf |
| | `design-api` | menambah/mengubah endpoint: kontrak, error, validasi, pagination, idempotency | draf |
| | `change-database` | menambah/mengubah tabel, index, migrasi, tipe uang/waktu | draf |
| | `build-backend` | logika server: lapisan, error, log, cache, antrean, rate limit | draf |
| UI | `setup-design-system` | menentukan token warna, font, spasi, tema proyek | draf |
| | `build-ui` | membuat komponen, halaman, form; responsif, aksesibilitas, SEO, performa | draf |
| | `manage-translations` | teks yang tampil ke user, menambah bahasa | draf |
| Fitur umum | `implement-auth` | daftar, login (email/OAuth/Telegram), sesi, reset, 2FA, peran, kunci API, JWT | draf |
| | `implement-payment` | gateway bayar, webhook, saldo/ledger, uang | draf |
| | `send-notifications` | email, Telegram, notifikasi dalam aplikasi | draf |
| | `build-admin-panel` | panel admin, audit log, konfirmasi aksi sensitif | draf |
| Kualitas & keamanan | `secure-feature` | fitur menyentuh login, uang, data pribadi, file, AI/LLM, pihak ketiga; bot/penyalahgunaan; menambah library | draf |
| | `write-tests` | menulis atau memperbaiki test | draf |
| | `write-docs` | README, cara menjalankan, dokumentasi API | draf |
| Infra | `deploy-app` | Docker, server, domain, HTTPS, CI/CD | draf |
| | `operate-app` | backup, pemantauan, log, alert, insiden | draf |

## Struktur repo

```
.claude-plugin/        manifest plugin + marketplace (Claude Code)
skills/<nama>/         SKILL.md + references/<stack>.md
templates/             kerangka skill dan file proyek
CATALOG.md             daftar alat/stack: status dan kapan dipakai
```

## Lisensi

MIT. Isi yang diadaptasi dari sumber lain mencantumkan sumbernya di bagian "Sumber" tiap skill.
