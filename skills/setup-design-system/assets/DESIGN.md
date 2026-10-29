# Desain: <nama proyek>

Sumber token di kode: `<path file token>`. Keputusan identitas: `docs/decisions/NNNN-…`.

## Identitas

Kesan: <2-3 kata>. Mode: terang / gelap / keduanya.

## Warna (token semantik)

| Token | Terang | Gelap | Untuk | Kontras (terang / gelap) |
|---|---|---|---|---|
| `background` | | | latar halaman | — |
| `surface` | | | kartu, panel | — |
| `text` | | | teks utama | x : 1 / x : 1 |
| `text-muted` | | | teks sekunder | |
| `border` | | | garis hiasan | |
| `border-strong` | | | batas input/kontrol (≥ 3 : 1) | |
| `primary` | | | aksi utama | |
| `primary-foreground` | | | teks di atas primary | |
| `danger` | | | error | |
| `success` | | | berhasil | |
| `warning` | | | peringatan | |
| `info` | | | informasi | |
| `focus` | | | garis fokus | |

## Tipografi

| Peran | Font | Ukuran | Tebal |
|---|---|---|---|
| Teks isi | | 16 px / 1.5 | |
| Judul halaman | | | |
| Angka/kode | (monospace) | | |

## Spasi, radius, bayangan, gerak

- Spasi: kelipatan 4 px.
- Radius: …
- Bayangan: …
- Gerak: … ms; mati saat `prefers-reduced-motion`.

## Ikon

Pustaka: …

## Komponen dasar

Diisi oleh `build-ui`.

| Komponen | Letak | Varian | Catatan |
|---|---|---|---|

## Aturan

- Komponen hanya memakai token semantik.
- Tidak ada warna/ukuran yang ditulis langsung.
- Token baru dicatat di sini dulu.
