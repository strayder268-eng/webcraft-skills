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

| Topik | Bentuk di proyek ini | Skill (aturan) | Status |
|---|---|---|---|
| contoh: letak komponen dasar | `web/components/ui/` | `structure-project` 2 | konvensi |

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

| Apa | Di mana | Siapa yang punya akses |
|---|---|---|
| contoh: HSTS, domain web | platform hosting | pemilik akun |
| contoh: DNS, SPF/DKIM/DMARC | penyedia DNS | pemilik akun |

## Lingkungan

| Lingkungan | Alamat | Cara deploy |
|---|---|---|
| Lokal | | |
| Produksi | | |

## Belum diketahui

Hal yang belum bisa dipastikan (mode proyek lama). Hapus bagian ini kalau kosong.
