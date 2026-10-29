---
name: secure-feature
description: Skill ini memeriksa keamanan sebelum dan sesudah coding - threat model ringan (STRIDE), checklist OWASP Top 10:2025, header keamanan dan CSP, file dan path traversal, SSRF, injeksi selain SQL, bot dan penyalahgunaan, fitur AI/LLM, pemilihan library, pemindaian rahasia, serta perlindungan data pribadi (UU PDP). Pakai saat fitur menyentuh login, uang, data pribadi, file, admin, model AI, atau layanan pihak ketiga, saat menambah atau memperbarui library, saat ada dugaan celah, atau sebelum rilis.
license: MIT
metadata:
  version: "0.5"
  language: id
  status: draf
---

# Mengamankan fitur

## Tujuan

Celah ditemukan saat masih berupa rencana (murah) dan bukan setelah dieksploitasi (mahal). Setiap risiko punya
keputusan yang tercatat: dicegah, dihilangkan, dialihkan, atau diterima dengan sadar.

## Kapan dipakai

- Fitur menyentuh login, uang, data pribadi, file, admin, atau pihak ketiga (dipanggil dari `build-feature`).
- Menambah/memperbarui library.
- Ada dugaan celah, atau menjelang rilis.

**Tidak dipakai untuk:** cara membuat login/pembayaran (`implement-auth`, `implement-payment`). Skill ini
memeriksa, skill itu membangun.

## Langkah

### 1. Threat model ringan [OWASP Threat Modeling]

Prioritas audit: P3.

Jawab empat pertanyaan, hasilnya ditulis di bagian "Keamanan" spek fitur:

1. **Apa yang dibangun?** Alur data fitur: dari mana masuk, lewat mana, disimpan di mana. Tandai **batas
   kepercayaan**: browser → API, API → layanan luar, layanan luar → webhook kita, admin → sistem.
2. **Apa yang bisa salah?** Pakai STRIDE pada setiap batas:

   | Ancaman | Pertanyaan |
   |---|---|
   | **S**poofing | bisakah seseorang mengaku sebagai user/sistem lain? |
   | **T**ampering | bisakah data diubah di jalan atau di klien (nominal, ID, peran)? |
   | **R**epudiation | bisakah aksi penting disangkal karena tidak tercatat? |
   | **I**nformation disclosure | bisakah data bocor (response, log, error, URL)? |
   | **D**enial of service | bisakah fitur dibanjiri sampai mati atau mahal (email, AI, SMS)? |
   | **E**levation of privilege | bisakah user biasa melakukan aksi admin/milik orang lain? |

   Pikirkan juga penyalahgunaan aturan bisnis: nominal negatif/nol, mengulang request, dua request bersamaan
   (race condition), melompati langkah alur.
3. **Apa yang dilakukan?** Untuk tiap ancaman: cegah / hilangkan / alihkan / terima (dengan alasan). Setiap
   pencegahan punya test.
4. **Sudah cukup?** Setelah dibangun, cek ulang daftar itu terhadap kode dan test.

### 2. Checklist OWASP Top 10:2025 [OWASP Top 10]

Prioritas audit: ikuti prioritas aturan skill yang dirujuk; P1 kalau tidak ada skill rujukan.

| Risiko | Cek | Skill terkait |
|---|---|---|
| A01 Broken Access Control | izin di server, cek pemilik objek, tolak bawaan | `design-api`, `implement-auth` |
| A02 Security Misconfiguration | mode debug mati, akun/kunci bawaan diganti, header keamanan, port tertutup | `deploy-app` |
| A03 Software Supply Chain Failures | library dicek, lockfile, CI terkunci | langkah 9 |
| A04 Cryptographic Failures | HTTPS di semua jalur, hash sandi benar, data rahasia dienkripsi, tidak membuat kripto sendiri | `implement-auth`, `change-database` aturan 11, `deploy-app` |
| A05 Injection | query berparameter, injeksi lain di langkah 6 | `change-database`, langkah 6 |
| A06 Insecure Design | threat model, rate limit, aturan bisnis dijaga server | langkah 1, `build-backend` |
| A07 Authentication Failures | sesi, sandi, 2FA, batas percobaan | `implement-auth` |
| A08 Software or Data Integrity Failures | tanda tangan webhook diverifikasi, image/artefak dari CI sendiri | `implement-payment`, `deploy-app` |
| A09 Security Logging and Alerting Failures | peristiwa penting dicatat, ada peringatan | `build-backend`, `operate-app` |
| A10 Mishandling of Exceptional Conditions | gagal tertutup, error tidak bocor, timeout | `build-backend` |

### 3. Header keamanan [OWASP HTTP Headers]

Prioritas audit: P1.

| Header | Nilai |
|---|---|
| `Strict-Transport-Security` | `max-age=63072000; includeSubDomains` (tambah `preload` hanya setelah yakin semua subdomain HTTPS) |
| `Content-Security-Policy` | kebijakan ketat berbasis nonce; tidak ada skrip inline tanpa nonce |
| `X-Content-Type-Options` | `nosniff` |
| `frame-ancestors` di CSP (atau `X-Frame-Options: DENY`) | `'none'` kecuali memang perlu di-embed |
| `Referrer-Policy` | `strict-origin-when-cross-origin` |
| `Permissions-Policy` | matikan fitur yang tidak dipakai: `camera=(), microphone=(), geolocation=()` |
| `Server`, `X-Powered-By` | dihapus |

`X-XSS-Protection` tidak dipasang (atau `0`). CSP baru dipasang dulu dalam mode `Report-Only`, diamati, lalu
ditegakkan.

Header boleh dipasang aplikasi, reverse proxy, atau platform hosting/CDN; yang wajib adalah header itu **sampai
ke browser**. Karena itu yang dicek adalah domain yang berjalan (`curl -I https://<domain>`), bukan hanya kode.
Kalau dipasang di platform, catat di *Dikelola di luar repo* `docs/PROJECT.md`.

### 4. File: upload, unduh, arsip [OWASP File Upload]

Prioritas audit: P1.

- Daftar jenis yang diizinkan (ekstensi **dan** isi file dicek), batas ukuran.
- Nama file diganti nama acak; tidak memakai nama/path dari user.
- Disimpan di penyimpanan file terpisah, bukan di folder yang dijalankan server.
- Disajikan dengan `Content-Disposition: attachment` atau dari domain terpisah; SVG/HTML dari user tidak
  disajikan inline.
- Metadata gambar (lokasi GPS) dibuang bila gambar ditampilkan ke orang lain.
- **Path traversal:** file diambil lewat ID yang dicari di database, bukan nama/path dari request. Kalau
  terpaksa memakai nama, buka hanya di dalam folder dasar dengan API yang menolak `..` dan symlink keluar
  (cara per bahasa di `references/`), dan cek pemiliknya.
- **Arsip (zip/gzip) dari user:** batasi jumlah file, total ukuran setelah dibuka, dan kedalaman; tolak nama
  entri berisi `..` atau path absolut (*zip slip*), supaya tidak bisa membanjiri disk (*zip bomb*).

### 5. Server mengambil URL dari user (SSRF) [OWASP SSRF]

Prioritas audit: P1.

Kalau server mengunduh/memanggil URL yang diberikan user (webhook user, pratinjau tautan, impor gambar):
domain diizinkan lewat daftar, alamat IP internal/privat ditolak **setelah** DNS di-resolve dan di setiap
redirect, batas waktu dan ukuran.

### 6. Injeksi selain SQL [OWASP XSS Prevention]

Prioritas audit: P1.

| Tempat | Aturan |
|---|---|
| HTML | biarkan framework meng-escape; HTML/Markdown dari user dibersihkan dengan sanitizer teruji sebelum ditampilkan |
| Tautan dari user | hanya skema `http:`/`https:` (tolak `javascript:`, `data:`) |
| URL yang disusun | setiap bagian di-encode (`encodeURIComponent`, `url.QueryEscape`) |
| Header HTTP/email | nilai dari user tidak boleh berisi `\r`/`\n`; pakai pustaka email, jangan menyusun header sendiri |
| Perintah sistem | hindari; kalau terpaksa, argumen terpisah tanpa shell (`references/go.md`) |
| Template | user tidak pernah bisa mengisi isi template yang dijalankan server |
| Regex | regex tidak pernah dari input user; di bahasa dengan regex *backtracking* (JavaScript), hindari pola bersarang seperti `(a+)+` yang bisa membuat server macet (ReDoS) |
| Log | log terstruktur (JSON) supaya baris baru dari input tidak bisa memalsukan baris log |

### 7. Bot dan penyalahgunaan [OWASP Automated Threats]

Prioritas audit: P1; batas biaya per user/kunci: P2.

- **Tantangan anti-bot** (mis. Turnstile/CAPTCHA) di daftar akun, lupa sandi, formulir publik, dan login
  setelah beberapa kali gagal. Token tantangan **diverifikasi di server**, bukan hanya di browser.
- **Bonus/kredit gratis/trial:** satu per identitas terverifikasi (email terverifikasi, akun Telegram/Google),
  tolak email sekali pakai, waspadai banyak akun dari IP/perangkat yang sama.
- **Biaya per user:** operasi yang mahal (panggilan AI, SMS, email, ekspor besar) punya kuota harian dan batas
  belanja per user/kunci, selain rate limit.
- **Sinyal kecurangan:** lonjakan pemakaian, banyak percobaan pembayaran gagal, pola akun baru yang sama →
  alert ke admin (`operate-app`), dan kemampuan membekukan akun.
- Abuse case ditulis bersama threat model: "bagaimana fitur ini bisa dipakai untuk merugikan kita/orang lain?"
  [OWASP Abuse Case]

### 8. Fitur AI/LLM [OWASP Top 10 for LLM Applications 2025]

Prioritas audit: P1.

Berlaku kalau fitur memanggil model AI (chat, ringkasan, agen, RAG), termasuk aplikasi yang meneruskan
permintaan ke penyedia model.

| Risiko | Aturan |
|---|---|
| LLM01 Prompt Injection | anggap **semua** isi yang masuk ke model bisa berisi perintah jahat, termasuk dokumen/halaman/email yang diambil (injeksi tidak langsung). Tidak bisa dicegah 100%, jadi batasi akibatnya (baris LLM06) |
| LLM02 Sensitive Information Disclosure | jangan mengirim data pribadi/rahasia yang tidak perlu ke penyedia model; cek kebijakan penyimpanan data penyedia; jawaban model tidak boleh berisi data user lain |
| LLM05 Improper Output Handling | keluaran model = **input tak tepercaya**: di-escape sebelum ditampilkan, divalidasi skemanya sebelum dipakai, tidak pernah langsung dijalankan sebagai SQL/perintah/kode/URL redirect |
| LLM06 Excessive Agency | alat (tools) yang bisa dipanggil model dibatasi seminimal mungkin, berjalan dengan izin user yang meminta (bukan izin admin), dan aksi yang mengubah data/uang/mengirim pesan butuh konfirmasi manusia |
| LLM07 System Prompt Leakage | anggap system prompt bisa bocor: tidak berisi kunci, sandi, atau aturan keamanan yang hanya dijaga lewat prompt |
| LLM08 Vector and Embedding Weaknesses | pencarian dokumen (RAG) memfilter berdasarkan izin user **sebelum** hasilnya diberikan ke model |
| LLM10 Unbounded Consumption | batasi panjang input, `max_tokens`, timeout, jumlah request, dan **biaya** per user/kunci; hentikan stream saat user memutus koneksi |

- Kunci API penyedia model hanya di server (terenkripsi kalau disimpan di database, `change-database`).
- Prompt dan jawaban yang dicatat di log mengikuti aturan data pribadi (langkah 11): seperlunya, dengan masa
  simpan.
- Model dan library AI melewati checklist library (langkah 9). Isi jawaban model bisa salah (LLM09): tampilkan
  penanda bahwa jawaban dibuat AI untuk hal yang berisiko.

### 9. Library [OWASP Vulnerable Dependency Management]

Prioritas audit: P1.

Sebelum menambah:

- [ ] Benar-benar perlu? Bisa dengan fitur bawaan bahasa/framework?
- [ ] Nama paket persis (waspada nama mirip/typosquatting).
- [ ] Masih dirawat (rilis terakhir, jumlah pengelola, isu keamanan yang terbuka).
- [ ] Lisensi cocok dengan proyek.
- [ ] Ukuran wajar (frontend: pengaruh ke ukuran JavaScript).
- [ ] Tidak ada kerentanan yang diketahui.

Selalu: lockfile di-commit, instal persis sesuai lockfile di CI, pemindai kerentanan di CI, pembaruan rutin,
kerentanan kritis ditambal secepatnya.

### 10. Rahasia

Prioritas audit: P1.

- Pemindai rahasia di CI (dan sebaiknya sebelum commit).
- Rahasia yang pernah ter-commit atau terkirim di chat **dianggap bocor**: ganti (rotasi) di penyedianya.
  Menghapusnya dari git tidak cukup.
- Rahasia berbeda untuk setiap lingkungan.

### 11. Data pribadi [UU PDP No. 27 Tahun 2022]

Prioritas audit: P1 untuk kebocoran data pribadi; P2 untuk masa simpan dan penghapusan.

- Kumpulkan **seperlunya**, dengan tujuan yang ditulis di kebijakan privasi.
- Persetujuan terpisah untuk pemasaran.
- User bisa melihat, mengunduh, memperbaiki, dan menghapus datanya (hapus akun).
- Masa simpan ditentukan; data yang tidak diperlukan dihapus.
- Kebocoran data pribadi: UU PDP mewajibkan pemberitahuan tertulis kepada pemilik data dan lembaga paling
  lambat 3 × 24 jam (Pasal 46). Siapkan prosedurnya di `operate-app`. Ini ringkasan, bukan nasihat hukum.

## Larangan

- Menganggap fitur aman karena "tidak ada yang tahu URL-nya".
- Keamanan yang hanya dijaga frontend.
- Menambah library tanpa checklist langkah 9.
- Menjalankan keluaran model AI tanpa validasi, atau memberi model alat dengan izin admin.
- Menunda rotasi rahasia yang bocor.

## Cek hasil

- [ ] Bagian "Keamanan" di spek berisi alur, batas kepercayaan, ancaman, dan keputusan.
- [ ] Setiap pencegahan punya test.
- [ ] Pemindai kerentanan dan pemindai rahasia lulus di CI.
- [ ] Header keamanan terpasang (cek dengan `curl -I`).
- [ ] Fitur AI: batas token/biaya per user ada dan diuji; keluaran model tidak dijalankan tanpa validasi.

## Stack

- `references/nextjs.md`: CSP dengan nonce, header di Next.js
- `references/go.md`: middleware header, `govulncheck`

## Sumber

- [OWASP Threat Modeling Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Threat_Modeling_Cheat_Sheet.html).
- [OWASP Top 10:2025](https://top10.owasp.org/2025).
- [OWASP HTTP Headers Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/HTTP_Headers_Cheat_Sheet.html).
- [OWASP Content Security Policy Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Content_Security_Policy_Cheat_Sheet.html).
- [OWASP File Upload Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html).
- [OWASP SSRF Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html).
- [OWASP Cross Site Scripting Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html).
- [OWASP: Regular expression Denial of Service (ReDoS)](https://owasp.org/www-community/attacks/Regular_expression_Denial_of_Service_-_ReDoS).
- [OWASP Automated Threats to Web Applications](https://owasp.org/www-project-automated-threats-to-web-applications/).
- [OWASP Abuse Case Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Abuse_Case_Cheat_Sheet.html).
- [OWASP Top 10 for LLM Applications 2025](https://genai.owasp.org/llm-top-10/).
- [OWASP LLM Prompt Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/LLM_Prompt_Injection_Prevention_Cheat_Sheet.html).
- [Cloudflare Turnstile: Server-side validation](https://developers.cloudflare.com/turnstile/get-started/server-side-validation/).
- [OWASP Vulnerable Dependency Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Vulnerable_Dependency_Management_Cheat_Sheet.html).
- [gitleaks](https://github.com/gitleaks/gitleaks): pemindai rahasia.
- [UU No. 27 Tahun 2022 tentang Pelindungan Data Pribadi](https://peraturan.bpk.go.id/Details/229798/uu-no-27-tahun-2022).
