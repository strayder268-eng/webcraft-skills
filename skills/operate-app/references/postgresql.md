# Backup PostgreSQL terenkripsi

Pola: `pg_dump` (format custom) → dienkripsi dengan **kunci publik GPG** admin → diunggah ke penyimpanan
objek (S3/R2/B2) dengan `rclone`. Server tidak pernah memegang kunci privat.

## Sekali di PC admin

```sh
gpg --quick-gen-key "app-backup" default default never
gpg --armor --export app-backup > backup-pubkey.asc                 # dikirim ke server
gpg --armor --export-secret-keys app-backup > app-backup-PRIVAT.asc  # simpan offline, JANGAN ke server
```

Di penyedia penyimpanan: buat bucket khusus backup dan kunci akses yang **hanya** bisa baca/tulis bucket itu.

## Skrip backup (server)

```sh
#!/usr/bin/env bash
# backup.sh: dijalankan cron, mis. setiap hari 02.15 waktu lokal
set -euo pipefail
cd "$(dirname "$0")"
set -a; . ./backup.env; set +a      # BUCKET, kredensial rclone (chmod 600, tidak di repo)

ts=$(date -u +%Y%m%dT%H%M%SZ)
out="backups/db-$ts.dump.gpg"
mkdir -p backups

docker compose exec -T db pg_dump -U app -d app -Fc </dev/null \
  | gpg --batch --yes --trust-model always --encrypt --recipient-file secrets/backup-pubkey.asc \
  > "$out"

sha256sum "$out" > "$out.sha256"
rclone copy "$out" "remote:$BUCKET/" && rclone copy "$out.sha256" "remote:$BUCKET/"

find backups -name 'db-*.dump.gpg*' -mtime +14 -delete                 # salinan lokal 14 hari
rclone delete "remote:$BUCKET/" --min-age 14d --include 'db-*'         # sesuaikan masa simpan
echo "ok $out"
```

- `</dev/null` pada `docker compose exec` penting: tanpa itu, kalau skrip dikirim lewat stdin (mis.
  `ssh … 'bash -s' < backup.sh`), `exec` ikut memakan sisa skrip dan langkah berikutnya diam-diam tidak jalan.
- `set -o pipefail` membuat skrip gagal kalau `pg_dump` gagal, bukan menghasilkan file kosong.
- Pantau: tulis waktu backup terakhir yang berhasil (atau kirim ping ke pemantau) dan beri alert kalau terlambat.

## Uji pulihkan (PC admin, bulanan)

```sh
#!/usr/bin/env bash
# restore-check.sh <file.dump.gpg>
set -euo pipefail
f="$1"
sha256sum -c "$f.sha256"
gpg --decrypt "$f" > /tmp/restore.dump

docker run -d --name restore-check -e POSTGRES_PASSWORD=t -e POSTGRES_USER=app -e POSTGRES_DB=app \
  postgres:16-alpine >/dev/null
trap 'docker rm -f restore-check >/dev/null; rm -f /tmp/restore.dump' EXIT
until docker exec restore-check pg_isready -U app -d app >/dev/null 2>&1; do sleep 1; done

docker exec -i restore-check pg_restore -U app -d app --no-owner --exit-on-error < /tmp/restore.dump
docker exec restore-check psql -U app -d app -c "SELECT count(*) AS users FROM users;"
echo "pulih OK"
```

Versi PostgreSQL penguji sama dengan produksi (atau lebih baru). Catat tanggal dan hasil di runbook.

## Sumber

- [PostgreSQL: pg_dump](https://www.postgresql.org/docs/current/app-pgdump.html)
- [PostgreSQL: pg_restore](https://www.postgresql.org/docs/current/app-pgrestore.html)
- [rclone: delete (--min-age)](https://rclone.org/commands/rclone_delete/)
- [GnuPG manual](https://www.gnupg.org/documentation/manuals/gnupg/)
