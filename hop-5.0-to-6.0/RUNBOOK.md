# Runbook — Hop 5.0 → 6.0

Langkah final yang terbukti benar, hasil saringan dari [NOTES.md](NOTES.md).

**Estimasi total waktu eksekusi:** ~5,9 jam (didominasi reindex ES ~5,6 jam — paling lama
dari semua hop sejauh ini). Lihat juga [../DOWNTIME_ESTIMATE.md](../DOWNTIME_ESTIMATE.md).

**PENTING — prasyarat Node.js:** hop ini butuh Node.js ≥16 untuk Vite (build tool JS baru
di Zammad 6.0). Dockerfile HARUS install Node.js via NodeSource (bukan `apt-get install
nodejs` biasa yang cuma dapat versi lama dari Debian Buster), plus `yarn`.

## Pre-flight

- [ ] Backup database, **verifikasi integritasnya** (`gzip -t` atau setara — JANGAN
  lanjut jika gagal). Detail lengkap: [../BACKUP_RESTORE.md](../BACKUP_RESTORE.md)
- [ ] Konfirmasi disk tersedia minimal 25GB bebas — image hop ini jauh lebih besar dari
  sebelumnya karena `node_modules` (build context ~1.1GB, vs puluhan-ratusan MB di hop lain)
- [ ] jika eksekusi ke produksi: window ~15 menit untuk build+migrate, reindex (~5,6 jam)
  bisa di background

## Langkah eksekusi

**1. Siapkan source code versi target** (tidak perlu patch gem — `bundle install` mulus)
```bash
git clone --branch 6.0.0 --depth 1 https://github.com/zammad/zammad.git app
```

**2. Build image** (Dockerfile final di folder ini — base `ruby:3.1.3-buster`, install
Node.js 18.x via NodeSource + yarn, `yarn install --frozen-lockfile` di build time,
Bundler `2.4.1`, `assets:precompile` tetap di runtime)
```bash
docker compose build
```

**3. Jalankan container**
```bash
docker compose up -d
```
Tunggu ~10-15 menit untuk `assets:precompile` (Sprockets + Vite build) selesai. Cek
`docker stats` jika ragu — CPU harus tinggi/aktif, bukan diam.

**4. Verifikasi koneksi database**
```bash
docker compose exec zammad-app env RAILS_ENV=production bundle exec rails runner \
  'puts ActiveRecord::Base.connection.execute("SELECT COUNT(*) FROM tickets").to_a'
```

**5. Migrasi schema** (~9,5 menit — jauh lebih lama dari hop lain, ada 2 migrasi berat)
```bash
docker compose exec zammad-app env RAILS_ENV=production bundle exec rake db:migrate
```

**6. Rebuild search index** (~5,6 jam — cek nama task dulu, sudah berubah lagi jadi
`zammad:searchindex:` dengan prefix)
```bash
docker compose exec zammad-app env RAILS_ENV=production bundle exec rake zammad:searchindex:rebuild
```
Jika gagal karena disk/index nyangkut, hapus manual dulu:
```bash
docker compose exec zammad-elasticsearch curl -X DELETE "http://localhost:9200/localhost.localdomain_zammad_production_*"
```

## Verifikasi akhir

- [ ] Admin Panel → System → Version menunjukkan "Zammad version 6.0.0"
- [ ] Search tiket berfungsi
- [ ] jika ada background job kustom yang bergantung ke `script/scheduler.rb` (misal
  systemd unit di produksi asli, bukan Docker) — update ke `script/background-worker.rb start`

## Jika perlu mundur (rollback)

Pola umum ada di [../BACKUP_RESTORE.md](../BACKUP_RESTORE.md) § "Pola umum rollback
per hop upgrade". Untuk hop ini: titik baginya adalah **tahap 5** — sebelum itu
tinggal hapus container/image (database masih di state hop 5.0), setelah itu wajib
restore dari backup pre-flight.

## Manajemen disk

Image hop ini jauh lebih besar dari hop-hop sebelumnya (node_modules), krisis disk
terjadi 2x — lihat [NOTES.md](NOTES.md). Urutan pembersihan aman standar ada di
[../ROADMAP.md](../ROADMAP.md) § "Pelajaran operasional lintas-hop" — jalankan lebih
sering selama proses debugging, jangan tunggu sampai kritis.
