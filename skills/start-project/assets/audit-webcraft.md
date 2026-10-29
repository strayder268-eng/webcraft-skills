# Audit webcraft-skills: <nama proyek>

Tanggal: YYYY-MM-DD
Versi skill: webcraft-skills <versi/commit>
Sifat: baca saja. Tidak ada kode yang diubah.

Putaran: 1 / 2 / gabungan
Yang tidak dijalankan (kalau ada larangan): …

## Ringkasan

| Prioritas | Jumlah | Inti masalah |
|---|---:|---|
| P1 keamanan dan rahasia | | |
| P2 data dan uang | | |
| P3 bug yang dirasakan pengguna | | |
| P4 struktur dan kerapian | | |
| **Total** | | |

## Temuan

Satu tabel per prioritas. ID tetap (`P1-01`, `P2-03`) supaya bisa dirujuk dari tugas perbaikan.

### P1: keamanan dan rahasia

| ID | Skill | Aturan | Bukti file:baris | Risiko | Status | Putaran |
|---|---|---|---|---|---|---|
| P1-01 | | | | | utang / perlu keputusan | 1+2 |

### P2: data dan uang

### P3: bug yang dirasakan pengguna

### P4: struktur dan kerapian

## Hasil penggabungan

Hanya di audit gabungan.

| | Jumlah |
|---|---:|
| Temuan di kedua putaran | |
| Hanya putaran 1, dicek ulang: nyata | |
| Hanya putaran 2, dicek ulang: nyata | |
| Dicoret (tidak nyata), dengan alasan di bawah | |
| Prioritas berbeda, diselesaikan dengan "Prioritas audit" | |

## Sumber bertentangan

Hasil `start-here` aturan 10: apa yang bertentangan, mana yang dipakai, dan temuan untuk sumber yang kalah.

| Topik | Sumber A | Sumber B | Yang benar (alasan) | Temuan |
|---|---|---|---|---|

## Perlu diukur

| Hal | Skill (aturan) | Cara mengukur |
|---|---|---|

## Yang sudah sesuai

- …

## Daftar periksa

Setiap aturan bernomor di setiap skill (selain skill alur kerja). Status: `✓` sesuai (wajib dengan bukti) ·
`T` temuan (sebut semua ID) · `K` konvensi · `U` perlu diukur · `–` tidak relevan (sebut alasan).

| Skill | Status per aturan |
|---|---|
| `structure-project` | 1 T(P4-01) · 2 K · 3 ✓ `web/components/ui/button.tsx:1-8` (hanya impor `lib/`) · 4 T(P4-02) · … |
| `change-database` | 7 T(P2-02, P2-05) · ✓ butir lain: `grep -n "ON DELETE" schema.sql` → 0 pada tabel ledger · … |
| `implement-payment` | – proyek tidak punya pembayaran |

## Cakupan dokumen

Semua file di `docs/` dan semua `*.md` lain di repo (kecuali `node_modules`, vendor, folder build).

| File | Hasil |
|---|---|
| | sesuai / usang (ID temuan) / riwayat / aset (tidak dibaca) |
