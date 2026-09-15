# Runbook — Hop 4.0 → 5.0

Langkah final yang terbukti benar, hasil saringan dari [NOTES.md](NOTES.md) (yang berisi
seluruh proses debugging, termasuk 3 bug versi-gap MariaDB yang akhirnya diatasi dengan
pindah database, bukan ditambal satu-satu).

**Estimasi total waktu eksekusi:** ~4,5 jam (didominasi reindex ES ~4,3 jam). Lihat juga
[../DOWNTIME_ESTIMATE.md](../DOWNTIME_ESTIMATE.md).

**PENTING — prasyarat database:** hop ini **WAJIB** jalan di atas MariaDB era 2021-2023
(10.6/10.11), BUKAN MariaDB 11.x. Jika dijalankan di MariaDB 11.x, kemungkinan besar akan
kena 3 bug yang didokumentasikan di NOTES.md #2 (parsing versi, BigDecimal, timestamp).
Jika ini dieksekusi ke **produksi sungguhan** nanti dan produksi masih di MariaDB 11.8.3,
**migrasi database dulu ke MariaDB 10.x (atau langsung ke PostgreSQL jika sudah di versi
Zammad ≥5.3) sebelum menjalankan hop ini** — jangan coba jalankan langsung di atas
MariaDB 11.x.

## Pre-flight

- [ ] Backup database (`mysqldump`), **verifikasi integritasnya** (`gzip -t` atau
  setara — JANGAN lanjut jika gagal). Detail lengkap: [../BACKUP_RESTORE.md](../BACKUP_RESTORE.md)
- [ ] Konfirmasi disk tersedia minimal 25GB bebas (hop ini paling boros disk dari semua
  hop sejauh ini — build cache + 2 database MariaDB berjalan bersamaan)
- [ ] jika eksekusi ke produksi: siapkan window ~10 menit untuk tahap 1-8 (build+migrate),
  reindex ES (~4 jam) bisa jalan di background setelah UI kembali bisa diakses

## Langkah eksekusi

**1. Siapkan source code versi target** (tidak perlu patch gem sama sekali di 5.0.0 —
mimemagic & tcr sudah dibenahi Zammad sendiri)
```bash
git clone --branch 5.0.0 --depth 1 https://github.com/zammad/zammad.git app
```

**2. Jika database masih MariaDB versi baru (11.x)** — siapkan dulu MariaDB 10.11
terpisah dan restore data ke situ (lihat `docker-compose.yml` di folder ini untuk service
`zammad-mariadb-legacy`, dan restore pakai `root`, BUKAN user aplikasi — dump berisi
trigger dengan `DEFINER` yang butuh privilege SUPER):
```bash
docker compose up -d zammad-mariadb-legacy
zcat backup.sql.gz | docker compose exec -T zammad-mariadb-legacy mariadb -u root -p'<root-password>' <nama_database>
```

**3. Build image** (Dockerfile final di folder ini — base `ruby:2.7.4-buster`, Bundler
`2.2.20`, `assets:precompile` di `command:` runtime bukan di Dockerfile — WAJIB karena
Rails 6.0 butuh koneksi database asli saat precompile, yang baru tersedia saat container
benar-benar jalan)
```bash
docker compose build
```

**4. Jalankan container**
```bash
docker compose up -d
```
Tunggu ~15-20 menit untuk `assets:precompile` selesai sebelum container `app` benar-benar
listening (normal, bukan hang — cek `docker stats` jika ragu, CPU harusnya tinggi/aktif).

**5. Verifikasi koneksi database**
```bash
docker compose exec zammad-app env RAILS_ENV=production bundle exec rails runner \
  'puts ActiveRecord::Base.connection.execute("SELECT COUNT(*) FROM tickets").to_a'
```

**6. Migrasi schema** (~1,5 menit)
```bash
docker compose exec zammad-app env RAILS_ENV=production bundle exec rake db:migrate
```

**7. Rebuild search index** (~4 jam)
```bash
docker compose exec zammad-app env RAILS_ENV=production bundle exec rake searchindex:rebuild
```
Jika gagal dengan `resource_already_exists_exception` (sisa index dari percobaan
sebelumnya), hapus manual dulu sebelum retry:
```bash
docker compose exec zammad-elasticsearch curl -X DELETE "http://localhost:9200/localhost.localdomain_zammad_production_*"
```

## Verifikasi akhir

- [ ] Admin Panel → System → Version menunjukkan "Zammad version 5.0.x"
- [ ] Search tiket berfungsi
- [ ] Console browser tidak ada error baru (warning CSP/notification permission yang
  sudah dikenal, aman diabaikan)

## Jika perlu mundur (rollback)

Pola umum ada di [../BACKUP_RESTORE.md § "Pola umum rollback
per hop upgrade"](../BACKUP_RESTORE.md#pola-umum-rollback-per-hop-upgrade). Untuk hop ini: titik baginya adalah **tahap 6** — sebelum itu tinggal
hapus container/image (database `zammad-mariadb-legacy` masih berisi data hasil
restore hop 4.0, belum ter-migrate, aman dipakai ulang), setelah itu wajib restore
dari backup pre-flight ke database legacy.

## Manajemen disk

Insiden disk penuh 2x terjadi di hop ini (100% penuh, 911MB tersisa — lihat
[NOTES.md](NOTES.md)). Urutan pembersihan aman standar ada di
[../ROADMAP.md § "Pelajaran operasional
lintas-hop"](../ROADMAP.md#pelajaran-operasional-lintas-hop-bukan-cuma-build-from-source) — jalankan
**sebelum** memulai hop berikutnya, jangan tunggu sampai disk kritis.
