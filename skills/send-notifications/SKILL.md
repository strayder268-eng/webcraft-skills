---
name: send-notifications
description: Skill ini mengirim notifikasi yang andal dan tidak mengganggu lewat email, bot Telegram, dan notifikasi dalam aplikasi - lewat antrean, template terjemahan, pengaturan preferensi dan berhenti berlangganan, keterkiriman email (SPF, DKIM, DMARC), batas kirim, tautan aman, dan masa simpan. Pakai saat menambah email (verifikasi, reset sandi, struk, pengumuman), pesan bot Telegram, lonceng notifikasi di aplikasi, atau webhook bot Telegram.
license: MIT
metadata:
  version: "0.1"
  language: id
  status: draf
---

# Mengirim notifikasi

## Tujuan

Pesan penting selalu sampai (tidak hilang, tidak masuk spam, tidak terkirim dua kali), pesan yang tidak penting
bisa dimatikan user, dan isi pesan tidak membocorkan data.

## Kapan dipakai

Email, pesan bot Telegram, notifikasi dalam aplikasi, webhook bot Telegram.

**Tidak dipakai untuk:** login dengan Telegram (`implement-auth`), antrean secara umum (`build-backend`).

## Data proyek yang dibutuhkan

- Penyedia email dan domain pengirim, bot Telegram yang dipakai (ADR).
- Daftar jenis notifikasi dan mana yang wajib (transaksional) vs bisa dimatikan.

## Aturan wajib

### 1. Jenis pesan

| Jenis | Contoh | Bisa dimatikan user? |
|---|---|---|
| Transaksional/keamanan | verifikasi email, reset sandi, struk, login baru, 2FA berubah | tidak |
| Info akun | saldo menipis, kunci API akan kedaluwarsa | ya |
| Pemasaran/pengumuman | promo, fitur baru | ya, **mati secara bawaan** sampai user setuju |

Preferensi disimpan per user per jenis per kanal.

### 2. Selalu lewat antrean

- Pengiriman tidak pernah dilakukan langsung di dalam request atau transaksi. Peristiwa disimpan di outbox
  bersama perubahan datanya, lalu worker mengirim (`build-backend` aturan 6).
- Setiap notifikasi punya kunci unik (mis. `receipt:payment:8412`) supaya tidak terkirim dua kali walau worker
  mengulang.
- Gagal sementara → coba lagi dengan jeda; gagal permanen (alamat tidak ada, bot diblokir) → berhenti dan catat.

### 3. Isi pesan

- Teks dari kamus terjemahan dengan bahasa user (`manage-translations`).
- **Tidak pernah** berisi sandi, kunci API lengkap, atau data pribadi yang tidak perlu.
- Kode OTP/verifikasi boleh, dengan masa berlaku singkat dan sekali pakai.
- Tautan memakai domain dari config; tautan yang memberi akses (reset, verifikasi, login) berisi token
  sekali pakai yang singkat masa berlakunya (`implement-auth`).
- Pesan keamanan menyertakan apa yang terjadi, kapan, dan apa yang harus dilakukan kalau bukan user sendiri.

### 4. Email: keterkiriman [Google: Email sender guidelines]

- Kirim lewat penyedia email transaksional (layanan SMTP/API), bukan dari server aplikasi langsung.
- Domain pengirim punya **SPF, DKIM (kunci ≥ 2048 bit), dan DMARC**, dengan domain `From:` selaras dengan SPF/DKIM.
- Pakai subdomain khusus (mis. `mail.example.com`) supaya reputasi pemasaran tidak merusak email transaksional.
- Email pemasaran: header **one-click unsubscribe** (`List-Unsubscribe` + `List-Unsubscribe-Post`) dan tautan
  berhenti berlangganan yang terlihat di isi. [RFC 8058]
- Tangani **bounce** dan **keluhan spam** dari penyedia: alamat yang bounce permanen atau mengeluh tidak dikirimi
  lagi.
- Jaga tingkat spam di bawah 0.3% (idealnya < 0.1%).
- Email berisi versi teks biasa selain HTML.

### 5. Telegram bot [Telegram Bot API]

- Bot hanya bisa mengirim ke user yang sudah memulai bot atau memberi izin (mis. scope
  `telegram:bot_access` saat login). Simpan `chat_id` per user.
- Token bot adalah rahasia server.
- Memakai webhook: daftarkan dengan `secret_token`, lalu **tolak** update yang tidak membawa header
  `X-Telegram-Bot-Api-Secret-Token` yang cocok.
- Batas kirim: ±1 pesan/detik per chat, 20 pesan/menit per grup, ±30 pesan/detik total. Respons 429 membawa
  `retry_after`; tunggu sesuai angka itu.
- User memblokir bot (403) → tandai kanal Telegram user itu nonaktif, jangan terus dicoba.

### 6. Notifikasi dalam aplikasi

- Tabel notifikasi: user, jenis, isi (kunci kamus + data), `read_at`, `created_at`.
- Daftar dengan pagination; tandai dibaca satu/semua.
- Masa simpan (mis. 90 hari) dan penghapusan terjadwal (`change-database`).
- User hanya bisa membaca notifikasinya sendiri (`design-api` aturan 7).

### 7. Batas dan biaya

- Batas kirim per user per jenis (mis. maksimal 1 email "saldo menipis" per hari).
- Fitur yang memicu pesan dari input orang lain (undangan, kirim ulang kode) diberi rate limit supaya tidak
  bisa dipakai untuk spam.

## Larangan

- Mengirim email/pesan langsung di dalam request.
- Pesan pemasaran tanpa persetujuan atau tanpa cara berhenti.
- Isi pesan dengan rahasia atau data sensitif.
- Mengabaikan bounce, keluhan, 429, dan 403.
- Webhook bot tanpa verifikasi `secret_token`.

## Cek hasil

- [ ] Test: notifikasi yang sama dipicu dua kali hanya terkirim sekali; preferensi "mati" dihormati.
- [ ] Domain email lulus SPF/DKIM/DMARC (cek header email uji: `spf=pass dkim=pass dmarc=pass`).
- [ ] Webhook bot menolak request tanpa secret yang benar.
- [ ] Template dicek di semua bahasa.

## Sumber

- [Google: Email sender guidelines](https://support.google.com/a/answer/81126).
- [RFC 8058: One-Click Unsubscribe](https://www.rfc-editor.org/rfc/rfc8058.html).
- [Telegram Bot API](https://core.telegram.org/bots/api): `setWebhook` `secret_token`, `retry_after`.
- [Telegram Bots FAQ](https://core.telegram.org/bots/faq): batas kirim.
