---
name: deploy-app
description: Skill ini menjalankan aplikasi di server dengan aman dan bisa diulang - persiapan server (SSH, firewall, update), image Docker yang dibuat CI, Docker Compose produksi, HTTPS otomatis lewat reverse proxy, domain/DNS, CI/CD (test, build, push image), urutan deploy dengan migrasi dan health check, rollback, dan rahasia. Pakai saat menyiapkan server baru, menulis Dockerfile/compose, mengatur domain atau HTTPS, membuat atau mengubah workflow CI/CD, melakukan deploy, atau rollback.
license: MIT
metadata:
  version: "0.4"
  language: id
  status: draf
---

# Deploy aplikasi

## Tujuan

Kode yang lulus test berubah menjadi aplikasi yang berjalan di server lewat langkah yang sama setiap kali,
tanpa rahasia bocor, dan bisa dikembalikan ke versi sebelumnya dalam hitungan menit.

## Kapan dipakai

Server baru, Dockerfile/compose, domain/HTTPS, CI/CD, deploy, rollback.

**Tidak dipakai untuk:** backup, pemantauan, insiden setelah aplikasi berjalan (`operate-app`).

## Data proyek yang dibutuhkan

- Tempat jalan (VPS/platform), domain, registry image (ADR, `docs/PROJECT.md`).
- **Deploy, perubahan DNS, dan perubahan server produksi selalu dengan izin user.**

## Aturan wajib

### 1. Build sekali, jalankan di mana saja [12-Factor: Build, release, run]

- Image dibuat **oleh CI** dari commit tertentu, diberi tag commit (`sha-1a2b3c4`), lalu disimpan di registry.
- Server tidak membangun kode dan tidak menyimpan kode sumber; server hanya menarik image.
- Image yang sama dipakai di staging dan produksi; yang berbeda hanya config (env).
- Tag image tidak pernah ditimpa isinya; versi lama tetap ada untuk rollback. Tag bergerak seperti `latest`
  boleh ada di registry, tapi **tidak dipakai** di compose/config produksi; produksi selalu menyebut tag
  commit.

### 2. Server dasar

- SSH hanya dengan kunci, login root dengan sandi dimatikan, akses SSH dibatasi IP bila memungkinkan.
- Firewall: yang terbuka ke publik hanya **443** (dan 80 kalau dibutuhkan untuk redirect/sertifikat).
  Database, Redis, panel internal **tidak pernah** terbuka ke internet.
- Update keamanan otomatis; jam server sinkron (NTP).

### 3. Image Docker [Docker best practices, OWASP Docker Security]

- *Multi-stage build*: alat build tidak ikut ke image akhir.
- Image dasar kecil dengan versi tertulis jelas (bukan `latest`).
- Berjalan sebagai **user non-root**.
- `.dockerignore` mengecualikan `.git`, `node_modules`, `.env*`, rahasia, folder build lokal.
- Rahasia tidak pernah masuk image (tidak lewat `COPY` dan tidak lewat build arg). Pengecualian: nilai yang
  memang publik dan dibekukan saat build (mis. `NEXT_PUBLIC_*`).
- Satu proses utama per container, dengan `HEALTHCHECK`.

### 4. Compose produksi

- Hanya reverse proxy yang mem-publish port. Layanan lain di jaringan internal.
- Setiap layanan: `restart: unless-stopped`, health check, batas memori, rotasi log.
- Rahasia dari file di server (Docker secrets / file `.env` ber-izin `600` di luar repo).
- Data permanen (database, sertifikat) di volume bernama.

### 5. HTTPS dan reverse proxy

- Sertifikat otomatis (mis. Caddy dengan Let's Encrypt). HTTP dialihkan ke HTTPS.
- `Strict-Transport-Security` dipasang (`secure-feature` aturan 3).
- Proxy meneruskan IP asli pengunjung dalam header; aplikasi hanya mempercayai header itu dari proxy sendiri.
- Batas ukuran body dan timeout di proxy.
- **API admin proxy dimatikan**, atau hanya bisa diakses dari dalam server (bukan dari container lain atau
  jaringan). Contoh bentuk: Caddy `{ admin off }` (`references/caddy.md`); nginx tidak punya API admin bawaan.
- Proxy/CDN tambahan di depan (mis. Cloudflare) dicatat sebagai ADR, termasuk dampaknya ke IP pengunjung dan
  latensi.

### 6. CI/CD [GitHub Actions: Security hardening]

Setiap push/PR:

1. typecheck + lint
2. test (dengan database test, `write-tests`)
3. pemindai kerentanan + pemindai rahasia (`secure-feature`)
4. build image

Di cabang utama: push image ke registry dengan tag commit.

Keamanan workflow:

- `permissions:` seminimal mungkin di setiap workflow.
- Action pihak ketiga dikunci ke **SHA commit**, bukan tag yang bisa dipindah.
- Input yang dikendalikan orang lain (judul PR, nama branch) tidak dimasukkan langsung ke perintah `run`.
- Rahasia CI hanya untuk workflow yang membutuhkannya.

### 7. Urutan deploy

1. Pastikan CI hijau untuk commit yang akan di-deploy.
2. **Backup database** (`operate-app`).
3. Jalankan migrasi tahap *expand* (`change-database`); kode lama harus tetap jalan dengan skema baru.
4. Tarik image versi baru, jalankan ulang layanan.
5. Tunggu health check hijau, lalu uji cepat (smoke test): halaman utama, login, satu endpoint API.
6. Gagal → **rollback**: jalankan lagi tag image sebelumnya. Karena migrasi bersifat expand, rollback aplikasi
   aman.
7. Catat versi yang berjalan (tag) dan waktu deploy.

Langkah *contract* (hapus kolom lama) baru dijalankan setelah versi baru stabil, di deploy berikutnya.

### 8. Lingkungan

- Minimal: lokal dan produksi. Staging disarankan begitu ada pembayaran atau user sungguhan.
- Rahasia, database, dan kunci layanan luar (mode sandbox vs live) **berbeda** di setiap lingkungan.

### 9. Konfigurasi aman di produksi [OWASP ASVS V13]

- Mode produksi aktif (`NODE_ENV=production`, mode debug framework mati); halaman error tidak menampilkan
  stack trace.
- Endpoint debug/profiling (mis. `pprof`, `/debug`, dashboard internal) mati, atau hanya bisa diakses dari
  jaringan internal.
- Source map JavaScript tidak dipublikasikan; folder `.git`, file `.env`, dan cadangan tidak pernah tersaji di
  web; daftar isi folder (*directory listing*) mati.
- Akun, sandi, dan kunci bawaan dari image/alat apa pun diganti sebelum online.
- Endpoint contoh/test dan data seed contoh tidak ikut ke produksi.
- Versi software tidak diumumkan di header/halaman error.

### 10. Koneksi antar layanan [OWASP ASVS V12]

- Satu server, satu jaringan Docker internal: koneksi ke database/Redis tanpa TLS dapat diterima karena tidak
  keluar dari mesin.
- Begitu melewati jaringan lain (database terkelola, server terpisah, antar region): **wajib TLS dengan
  verifikasi sertifikat** (PostgreSQL `sslmode=verify-full`, Redis TLS).
- Panggilan keluar ke layanan luar selalu memverifikasi sertifikat; mematikan verifikasi
  (`InsecureSkipVerify`, `NODE_TLS_REJECT_UNAUTHORIZED=0`) dilarang di luar test lokal.

### 11. Efisiensi resource

- **Build cepat:** urutan layer dari yang jarang berubah ke yang sering berubah (salin file dependensi →
  instal → baru salin kode), cache build di CI, dan cache unduhan dependensi di dalam build
  (`references/docker.md`). Mengubah satu baris kode tidak boleh mengunduh ulang semua dependensi.
- **Image kecil:** multi-stage, image dasar kecil, tanpa alat build. Ukuran image dicatat; kenaikan besar
  diperiksa sebabnya.
- **Batas CPU/memori dari hasil ukur**, bukan tebakan: lihat pemakaian nyata (`docker stats`) saat beban biasa
  dan puncak, beri ruang ±30-50%, lalu pasang batasnya. Container yang kena batas memori (OOM) harus terlihat di
  pemantauan (`operate-app`).
- **Pool dan worker sesuai kapasitas:** jumlah koneksi pool database semua instance ditotal tidak melebihi
  `max_connections` database dikurangi cadangan untuk admin/migrasi.
- **Disk:** rotasi log container, hapus image lama terjadwal tapi sisakan beberapa versi terakhir untuk
  rollback.

## Larangan

- Build atau edit kode langsung di server produksi.
- Deploy tanpa backup atau tanpa tahu tag versi sebelumnya.
- Port database/Redis terbuka ke publik.
- Container berjalan sebagai root tanpa alasan tercatat.
- Rahasia di image, di repo, atau di log CI.
- Action CI pihak ketiga tanpa dikunci SHA.

## Cek hasil

- [ ] Port yang terbuka dari luar hanya yang direncanakan (cek dari luar server).
- [ ] `curl -I https://domain` menunjukkan HTTPS valid dan header keamanan.
- [ ] Rollback sudah pernah dicoba (minimal sekali di staging/lokal).
- [ ] Dari luar: `/.git/`, `/.env`, endpoint debug, dan file `.map` mengembalikan 404.
- [ ] Build ulang setelah mengubah satu file kode tidak mengunduh ulang dependensi (cache bekerja).
- [ ] Tidak ada rahasia di image (`docker history`, isi image) maupun di log CI.

## Stack

- `references/docker.md`: Dockerfile Go dan Next.js, compose produksi
- `references/github-actions.md`: workflow CI dan build image
- `references/caddy.md`: reverse proxy HTTPS

## Sumber

- [The Twelve-Factor App: Build, release, run](https://12factor.net/build-release-run).
- [Docker: Building best practices](https://docs.docker.com/build/building/best-practices/).
- [OWASP Docker Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html).
- [GitHub Actions: Security hardening](https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions).
- [Caddy: Automatic HTTPS](https://caddyserver.com/docs/automatic-https).
- [OWASP ASVS 5.0](https://owasp.org/www-project-application-security-verification-standard/): V12 Secure Communication, V13 Configuration.
- [PostgreSQL: SSL support (sslmode)](https://www.postgresql.org/docs/current/libpq-ssl.html).
- [Next.js: Self-hosting](https://nextjs.org/docs/app/guides/self-hosting).
