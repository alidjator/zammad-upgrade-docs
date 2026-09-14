# Runbook — Hop 6.0 → 7.0

Langkah eksekusi terencana (belum divalidasi end-to-end di server — update runbook
ini kalau ada penyesuaian saat eksekusi nyata, ikuti pola runbook hop-hop sebelumnya).

**Prasyarat:** hop 5.0→6.0 sudah selesai & tervalidasi, migrasi PostgreSQL sudah
selesai & tervalidasi (staging sudah berjalan di atas PostgreSQL host, bukan MariaDB).

## Pre-flight

- [ ] Cek disk space dulu (`df -h`, `docker system df`) — rutin jadi sumber masalah di
      tiap hop sebelumnya. Bersihkan build cache (`docker builder prune -af`) kalau perlu.
- [ ] Backup database PostgreSQL staging (`pg_dump`) sebelum mulai — jaring pengaman
      independen dari `zammad-mariadb-legacy` yang sudah ada.

## Langkah eksekusi

**1. Clone source Zammad 7.0.0 di server**
```bash
cd /usr/local/src/zammad-staging
rm -rf app
git clone --branch 7.0.0 --depth 1 https://github.com/zammad/zammad.git app
```

**2. Sync file Dockerfile/Dockerfile.elasticsearch/docker-compose.yml/database.yml
ke server** (dari folder `hop-6.0-to-7.0/` ini — JANGAN lupa langkah ini, pernah
kelewat di hop 5.0→6.0 dan menyebabkan build pakai file basi)

**3. Uji `bundle install` + `pnpm install` di container sementara dulu** (pola yang
sudah terbukti efektif — tangkap masalah gem/paket sebelum build image penuh)
```bash
docker run --rm -v "$(pwd)/app:/opt/zammad" -w /opt/zammad ruby:3.4.8-bookworm bash -c "
  apt-get update && apt-get install -y curl gnupg git build-essential libpq-dev &&
  curl -fsSL https://deb.nodesource.com/setup_20.x | bash - &&
  apt-get install -y nodejs &&
  corepack enable &&
  pnpm install --frozen-lockfile &&
  gem install bundler -v '2.6.9' &&
  bundle install --without development test
"
```
Kalau `apt-get update` gagal karena Bookworm sudah EOL saat eksekusi nyata, terapkan
workaround `archive.debian.org` yang sama seperti hop 1-3.

**4. Build image (jalankan di dalam `screen` — proses build + assets:precompile lama)**
```bash
screen -S hop7-build
docker compose build
docker compose up -d
# Ctrl+A lalu D untuk detach, screen -r hop7-build untuk reattach cek progress
```

**5. Migrasi database Rails**
```bash
docker compose exec zammad-app env RAILS_ENV=production bundle exec rake db:migrate
```

**6. Rebuild search index (WAJIB di hop ini — perubahan ASCII-folding, beda dari
migrasi Postgres kemarin yang tidak butuh ini)**
```bash
docker compose exec zammad-app bundle exec rake --tasks | grep -i -E "index|search"
# pakai nama task yang muncul dari output di atas (sudah berubah nama beberapa kali antar versi)
docker compose exec zammad-app env RAILS_ENV=production bundle exec rake <nama_task_searchindex_rebuild>
```

**7. Verifikasi**
```bash
docker compose exec zammad-app env RAILS_ENV=production bundle exec rake db:migrate:status | grep -v "^   up"
docker compose logs zammad-app --tail 100
```
- [ ] UI menampilkan versi 7.0.x
- [ ] Login & buka tiket normal
- [ ] Search tiket berfungsi (validasi hasil rebuild index)
- [ ] WebSocket real-time update jalan (cek `/cable` di browser devtools)

## Kalau gagal / perlu mundur

`zammad-mariadb-legacy` tidak lagi dipakai service manapun di hop ini, tapi datanya
masih utuh sebagai jaring pengaman terakhir kalau perlu mundur jauh ke sebelum
migrasi Postgres. Untuk mundur satu langkah (ke image 6.0.0), cukup checkout ulang
folder `app/` ke tag `6.0.0` dan rebuild — data PostgreSQL tidak berubah struktur
secara merusak oleh migrasi Rails yang gagal di tengah jalan (migrasi Rails idempoten
per-file, cek `db:migrate:status` untuk tahu titik terakhir yang berhasil).
