# <Nama proyek>

## Tujuan

Untuk apa web ini dan untuk siapa. 2-3 kalimat.

## Pengguna

| Peran | Bisa melakukan |
|---|---|
| Pengunjung | … |
| Pengguna terdaftar | … |
| Admin | … |

## Stack

| Bagian | Teknologi | Versi |
|---|---|---|
| Frontend | | |
| Backend | | |
| Database | | |
| Cache/antrean | (tidak ada / Redis) | |
| Infra | | |

Keputusan stack: `docs/decisions/0001-…` (proyek lama: rujuk dokumen keputusan yang sudah ada, atau tulis
"belum ada ADR").

## Struktur

Folder utama dan isinya (ringkas). Detail aturan: skill `structure-project`.

## Konvensi

Bentuk yang dipakai proyek ini bila berbeda dari "Contoh bentuk" di skill. Status: `konvensi` (dipakai
terus) / `utang` (melanggar aturan, dirapikan bertahap) / `perlu keputusan`.

| Topik | Bentuk di proyek ini | Skill (aturan) | Status | Temuan |
|---|---|---|---|---|
| contoh: letak komponen dasar | `web/components/ui/` | `structure-project` 2 | konvensi | – |
| contoh: tombol di halaman lama | `<button>` mentah | `build-ui` 1 | utang | P4-05 |
| contoh: migrasi | satu `schema.sql` dimuat ulang | `change-database` 8 | perlu keputusan | P2-01 |

## Target

| Ukuran | Target | Sumber |
|---|---|---|
| p95 endpoint baca / tulis | 200 ms / 500 ms | bawaan `build-backend` 12 |
| RPO / RTO | | `operate-app` |

## Layanan luar

| Layanan | Untuk apa | Env yang dipakai |
|---|---|---|

## Menjalankan

```sh
# perintah menjalankan lokal
# perintah test
```

## Dikelola di luar repo

Konfigurasi yang tidak ada di repo/server sendiri. Audit menandainya "cek manual oleh user".

| Apa | Di mana | Siapa yang punya akses | Cara cek | Terakhir dicek |
|---|---|---|---|---|
| contoh: HSTS, domain web | platform hosting | nama orang/akun | `curl -I https://<domain>` | YYYY-MM-DD |
| contoh: DNS, SPF/DKIM/DMARC | penyedia DNS | nama orang/akun | `dig TXT <domain>` | |
| contoh: env produksi frontend | dasbor platform hosting | nama orang/akun | daftar nama env (tanpa nilai) | |

## Lingkungan

| Lingkungan | Alamat | Cara deploy |
|---|---|---|
| Lokal | | |
| Produksi | | |

## Belum diketahui

Hal yang belum bisa dipastikan (mode proyek lama). Hapus bagian ini kalau kosong.
