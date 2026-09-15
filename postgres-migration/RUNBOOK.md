# Runbook — Migrasi MariaDB → PostgreSQL

Langkah final yang terbukti benar, hasil saringan dari [NOTES.md](NOTES.md).

**Estimasi total waktu eksekusi:** ~50 menit (43 menit pgloader + overhead backup/restart).
Ini migrasi database, BUKAN hop upgrade Zammad — tidak ada `assets:precompile`/reindex ES.

**Prasyarat:** Zammad harus sudah ≥5.3 (tool `rake zammad:db:pgloader` baru ada mulai
versi itu). Kalau eksekusi ke produksi nanti, pastikan PostgreSQL target sudah disiapkan
dan reachable dari container SEBELUM memulai.

## Pre-flight

- [ ] PostgreSQL target sudah terpasang, versi ≥13 (dicek: `psql --version`)
- [ ] Buat database + user terisolasi khusus (jangan reuse punya aplikasi lain kalau
  instance PostgreSQL dipakai bersama)
- [ ] Konfirmasi `listen_addresses` dan `pg_hba.conf` mengizinkan koneksi dari network Docker
- [ ] Backup MariaDB source, **verifikasi integritas backup** (`gzip -t`) sebelum lanjut

## Langkah eksekusi

**1. Buat database + user PostgreSQL terisolasi**
```bash
sudo -u postgres psql << 'EOF'
CREATE USER <user_staging> WITH PASSWORD '<password>';
CREATE DATABASE <db_staging> OWNER <user_staging> ENCODING 'UTF8';
GRANT ALL PRIVILEGES ON DATABASE <db_staging> TO <user_staging>;
EOF
```

**2. Backup MariaDB source (WAJIB verifikasi integritas)**
```bash
docker compose exec <service_mariadb> mariadb-dump -u root -p'<root_password>' <database> | gzip > backup.sql.gz
gzip -t backup.sql.gz && echo "GZIP OK"   # JANGAN lanjut kalau ini gagal
```

**3. Tambah `extra_hosts` ke docker-compose.yml** (kalau PostgreSQL di host, bukan container)
```yaml
extra_hosts:
  - "host.docker.internal:host-gateway"
```
Terapkan ke `zammad-app`, `zammad-websocket`, `zammad-scheduler`, lalu `docker compose up -d`.

**4. Generate & edit command pgloader**
```bash
docker compose exec zammad-app env RAILS_ENV=production bundle exec rake zammad:db:pgloader > pgloader-command
sed -i "s|pgsql://zammad:pgsql_password@localhost/zammad|pgsql://<user_staging>:<password>@host.docker.internal/<db_staging>|" pgloader-command
```

**5. Dry-run dulu** (cek koneksi source & target OK sebelum commit ke migrasi sungguhan)
```bash
docker run --rm --network <nama_network_compose> --add-host host.docker.internal:host-gateway \
  -v "$(pwd)/pgloader-command:/pgloader-command" dimitri/pgloader pgloader --dry-run /pgloader-command
```

**6. Jalankan migrasi sungguhan — WAJIB di dalam `screen`/`tmux`** (proses lama, ~40+ menit,
jangan sampai terputus kalau koneksi SSH bermasalah)
```bash
screen -S pgloader
docker run --rm --network <nama_network_compose> --add-host host.docker.internal:host-gateway \
  -v "$(pwd)/pgloader-command:/pgloader-command" dimitri/pgloader pgloader --verbose /pgloader-command
# Ctrl+A lalu D untuk detach, screen -r pgloader untuk reattach cek progress
```

**7. Validasi row count** (source vs target, tabel-tabel utama) — jangan lanjut ke
langkah 8 kalau ada yang tidak cocok.

**8. Ubah `database.yml` ke adapter `postgresql`, restart**
```bash
docker compose restart zammad-app zammad-websocket zammad-scheduler
```

**9. Verifikasi**
```bash
docker compose exec zammad-app env RAILS_ENV=production bundle exec rails runner \
  'puts ActiveRecord::Base.connection.adapter_name'
docker compose exec zammad-app env RAILS_ENV=production bundle exec rake db:migrate:status | grep -v "^   up"
```
(command kedua harus TIDAK menampilkan baris apa pun selain header — kalau ada baris
`down`, ada migrasi yang belum jalan)

## Verifikasi akhir

- [ ] `adapter_name` = "PostgreSQL"
- [ ] `db:migrate:status` semua `up`
- [ ] UI & search berfungsi normal

## Kalau perlu mundur (rollback)

Selama `database.yml` masih bisa dikembalikan ke config MariaDB lama DAN container
MariaDB source belum dihapus — tinggal revert `database.yml`, restart. Data MariaDB
tidak tersentuh sama sekali oleh proses pgloader (pgloader cuma READ dari source).

## Setelah stabil beberapa hari

Baru pertimbangkan matikan/hapus container MariaDB source untuk membebaskan resource —
jangan buru-buru, ini jaring pengaman rollback termudah selama masa transisi.

**Update 15 Sept 2026** — container sudah di-`stop` (bukan dihapus) saat disk mepet di
hop 7.0→7.1.3. Seluruh proyek upgrade (sampai 7.1.3) sudah selesai & tervalidasi di
atas PostgreSQL lewat 2 hop major version penuh dengan trafik nyata, tanpa insiden
integritas data — tapi secara kalender baru ~4 hari sejak migrasi. **Keputusan
eksplisit user: tunggu beberapa hari lagi (bukan hapus sekarang)** sebelum
mempertimbangkan hapus total container+volume (~6,7GB). Catatan penting: 2 file dump
backup independen (`.sql.gz`) sudah dihapus minggu ini dengan asumsi volume ini tetap
ada sebagai satu-satunya salinan — begitu diputuskan hapus nanti, itu berarti tidak
ada lagi salinan MariaDB pra-migrasi sama sekali (Zammad 7.0+ juga sudah tidak
mendukung MySQL, jadi nilai rollback data ini sudah sangat menurun).
