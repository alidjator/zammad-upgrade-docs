# Runbook — Hop 7.0 → 7.1.3 (hop terakhir)

Rencana eksekusi berdasarkan riset requirement + seluruh pelajaran dari hop 1-6
sebelumnya. Perbarui bagian yang berubah setelah eksekusi nyata.

**Prasyarat:** hop 6.0→7.0 sudah selesai & tervalidasi (tag `hop-6.0-to-7.0-done`).

## Pre-flight

- [ ] Cek disk space (`df -h /`, `docker system df`) — bersihkan dulu jika di bawah
      ~15GB tersisa. Riwayat proyek ini menunjukkan krisis disk terjadi di HAMPIR
      SETIAP hop — jangan asumsikan aman.
- [ ] Backup database PostgreSQL staging (`pg_dump`), verifikasi integritasnya
      (jangan cuma asumsikan sukses dari tidak adanya error). Detail lengkap:
      [../BACKUP_RESTORE.md](../BACKUP_RESTORE.md)

## Langkah eksekusi

**1. Clone source Zammad 7.1.3**
```bash
cd /usr/local/src/zammad-staging
rm -rf app
git clone --branch 7.1.3 --depth 1 https://github.com/zammad/zammad.git app
```

**2. Sync `Dockerfile`, `Dockerfile.elasticsearch`, `docker-compose.yml` dari folder
`hop-7.0-to-7.1.3/` ini ke server.** Perhatikan 2 hal yang WAJIB benar di Dockerfile:
- NodeSource `setup_24.x` (bukan `setup_20.x` lagi)
- `pkg-config` + `libimlib2-dev` + `ENV CI=true` tetap ada (jangan sampai lupa
  meng-copy dari template, ini penyebab 2 insiden nyata di hop sebelumnya)

Dan di `docker-compose.yml`: pastikan `zammad-app`/`websocket`/`scheduler` semua
punya `image: zammad-staging-app:7.1.3` yang SAMA.

**3. Uji `bundle install` + `pnpm install` di container sementara**
```bash
docker run --rm -v "$(pwd)/app:/opt/zammad" -w /opt/zammad ruby:3.4.9-bookworm bash -c "
  apt-get update && apt-get install -y curl gnupg git build-essential libpq-dev \
    imagemagick poppler-utils ca-certificates shared-mime-info pkg-config libimlib2-dev &&
  curl -fsSL https://deb.nodesource.com/setup_24.x | bash - &&
  apt-get install -y nodejs &&
  corepack enable &&
  CI=true pnpm install --frozen-lockfile &&
  gem install bundler -v '2.6.9' &&
  bundle install --without development test
"
```
Harus selesai dengan `Bundle complete!` tanpa error.

**4. Build image — SATU service saja, lalu `up -d` LANGSUNG**
```bash
screen -S hop7.1.3-build
docker compose build zammad-app 2>&1 | tee build-hop7.1.3.log
docker compose up -d
```
⚠️ **Jangan** jalankan `docker image prune -af` di antara `build` dan `up -d` — pernah
menghapus image yang baru dibuild karena dianggap "belum terpakai".

**5. Verifikasi container stabil**
```bash
docker compose ps
docker compose logs zammad-app --tail 50
```

**6. Migrasi database**
```bash
docker compose exec zammad-app env RAILS_ENV=production bundle exec rake db:migrate
```
Jika ada migrasi baru dengan pola bug urutan seperti `recent_closes` di hop
sebelumnya (migrasi lama tiba-tiba butuh tabel yang dibuat migrasi jauh lebih baru),
lihat [hop-6.0-to-7.0/NOTES.md](../hop-6.0-to-7.0/NOTES.md) Insiden 6 untuk pola
diagnosis & fix-nya (`db:migrate:up VERSION=<versi_migrasi_pendukung>` duluan, verifikasi
migrasi itu berdiri sendiri tanpa dependency ke migrasi lain, baru lanjut `db:migrate`
normal).

Ambil durasi migrasi:
```bash
docker compose exec zammad-app grep "Migrating to" log/production.log | head -1
docker compose exec zammad-app grep "Migrating to" log/production.log | tail -1
docker compose exec zammad-app env RAILS_ENV=production bundle exec rake db:migrate:status | grep -v "^   up"
```

**7. Verifikasi asset pipeline benar-benar ter-precompile**
```bash
docker compose exec zammad-app ls public/assets/ | grep -E "application-.*\.css"
curl -sI http://localhost:3010/
```
Jika 500 atau file CSS tidak ada, jalankan precompile manual + restart (lihat
[hop-6.0-to-7.0/RUNBOOK.md](../hop-6.0-to-7.0/RUNBOOK.md) langkah 7 untuk detail).

**8. Rebuild search index — cek index stale dulu**
```bash
docker compose exec zammad-elasticsearch curl -s "http://localhost:9200/_cat/indices?v"
```
Jika ada index `zammad_production_*` yang seharusnya sudah tidak ada, hapus dulu
(`DELETE /localhost.localdomain_zammad_production_*`). Baru jalankan rebuild:
```bash
screen -S hop7.1.3-reindex
docker compose exec zammad-app env RAILS_ENV=production bundle exec rake zammad:searchindex:rebuild 2>&1 | tee reindex-hop7.1.3.log
```
**Pantau disk selama proses ini berjalan** (`df -h /` sesekali) — jika mendekati 95%,
langsung bersihkan dan hapus block ES (`PUT _all/_settings
{"index.blocks.read_only_allow_delete": null}`) sebelum disk benar-benar penuh,
jangan tunggu sampai proses gagal (pelajaran Insiden 9).

**9. Verifikasi akhir**
- [ ] UI menampilkan versi 7.1.3
- [ ] Login & buka tiket normal
- [ ] Search tiket berfungsi (query langsung ke ES jika mau verifikasi lebih pasti:
  `GET _search` untuk nomor tiket spesifik, bandingkan `_count` dengan jumlah tiket asli)
- [ ] WebSocket real-time update jalan

## Setelah hop ini selesai — proyek upgrade LENGKAP

Ini hop terakhir di roadmap. Setelah tervalidasi:
- [ ] Update README.md status table, buat tag `hop-7.0-to-7.1.3-done`
- [ ] Diskusikan dengan user: kapan mempertimbangkan mematikan `zammad-mariadb-legacy`
      (sudah tidak dipakai sejak hop 6.0→7.0, dibiarkan sebagai rollback safety net)
- [ ] Rujuk ke [PRODUCTION_READINESS_TODO.md](../PRODUCTION_READINESS_TODO.md) —
      gap analysis kesiapan cutover produksi asli, belum dikerjakan, perlu dibahas
      sebelum domain `helpdesk.satu.solutions` benar-benar dialihkan ke `zammad-audit`
      yang sudah di-upgrade
