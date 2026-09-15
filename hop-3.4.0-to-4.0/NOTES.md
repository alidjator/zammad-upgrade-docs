# Hop 3.4.0 → 4.0 — Catatan Lengkap (Status: ✅ Selesai & Tervalidasi)

## Ringkasan

Database MariaDB (`zammad_staging`, restore dari backup produksi 3.4.0) → migrasi schema
sukses ke skema 4.0 → UI tervalidasi berjalan normal di `helpdesk.satu.solutions` dengan
data asli produksi.

## Environment

- Server: CentOS Stream 9 (`Koi-Server-Dev`)
- Staging project: `/usr/local/src/zammad-staging` (Docker Compose project name `zammad-staging`)
- Database: MariaDB `11.8.3` di host (`host.docker.internal` dari sudut pandang container),
  database `zammad_staging`, user `zammad_staging` (privilege HANYA ke `zammad_staging.*`)
- Sumber Zammad 4.0.0: `git clone --branch 4.0.0 --depth 1 https://github.com/zammad/zammad.git app`

## Insiden 1 — `mysql`/`mysqldump` client tidak ditemukan di container MariaDB
MariaDB 10.6+/11.x mengganti nama binary client dari `mysql` → `mariadb` (rebranding).
**Fix:** pakai `mariadb` bukan `mysql` saat exec ke container MariaDB.

## Insiden 2 — Bash history expansion pada password mengandung `!`
`-p"pass!word"` di dalam double-quote masih diproses bash sebagai history substitution.
**Fix:** selalu pakai single-quote untuk password: `-p'pass!word'`.

## Insiden 3 — Password salah: pakai password `zammad_audit` untuk login `root`
Contoh compose yang dipakai punya 2 password berbeda (root vs user aplikasi) — tertukar saat
testing manual. **Fix:** selalu double-check password mana yang dipakai untuk user mana.

## Insiden 4 — `mimemagic (0.3.5)` sudah di-yank dari rubygems.org
Build gagal di `bundle install` karena versi lama sudah dihapus total dari RubyGems (masalah
lisensi GPL). `bundle lock --update mimemagic --conservative` juga GAGAL (Bundler 1.17.3 tidak
bisa "melompat" dari versi yang sudah sama sekali tidak bisa di-fetch).

**Fix — edit `Gemfile.lock` langsung:**
```bash
sed -i 's/^    mimemagic (0.3.5)$/    mimemagic (0.3.10)/' app/Gemfile.lock
sed -i '/^    mimemagic (0.3.10)$/a\      nokogiri (~> 1)\n      rake' app/Gemfile.lock
```
(dependency 0.3.10 dikonfirmasi via `https://rubygems.org/api/v2/rubygems/mimemagic/versions/0.3.10.json`
→ butuh `nokogiri (~> 1)` dan `rake`)

## Insiden 5 — Gem `tcr` — git commit sudah hilang dari repo `zammad-deps/tcr`
```
fatal: Could not parse object 'ddc8caf9d57a991c8af850d2870969e7a265ec59'.
```
Repo `zammad-deps/tcr` sudah di-reset/rewrite historinya sejak dulu, commit yang dikunci di
lock file sudah tidak ada. Dicek: `tcr` cuma dipakai di grup `test` (bareng `vcr`, untuk
merekam HTTP request saat testing) — TIDAK dibutuhkan runtime produksi.

**Fix:**
```bash
sed -i "/gem 'tcr', git:/d" app/Gemfile
# lalu regenerate lock supaya konsisten (jalankan di container sementara, lihat di bawah)
bundle lock --conservative
```

## Insiden 6 — Proses fix di atas dijalankan via container sementara (bukan langsung di Dockerfile)
Supaya `Gemfile`/`Gemfile.lock` yang sudah diperbaiki **persisten di disk** (bukan hilang tiap
build ulang), semua fix dijalankan lewat container temporer dengan bind-mount:
```bash
docker run --rm -v /usr/local/src/zammad-staging/app:/opt/zammad -w /opt/zammad ruby:2.6.6-stretch bash -c "
  <setup apt archive.debian.org sama seperti Dockerfile> &&
  gem install bundler -v '~> 1.17' &&
  bundle lock --conservative &&
  bundle lock --update mimemagic --conservative &&
  bundle install --full-index --without development test
"
```
Setelah `Bundle complete!` muncul di container sementara ini, baru `docker compose build` di
compose project yang sebenarnya (hasilnya sukses karena Gemfile.lock sudah bersih).

## Insiden 7 — MariaDB user hanya bisa login dari `localhost`, bukan dari IP container
```
Access denied for user 'zammad_staging'@'172.30.0.x' (using password: YES)
```
User awalnya cuma ke-grant untuk host tertentu (bukan `%`). **Fix:**
```sql
CREATE USER IF NOT EXISTS 'zammad_staging'@'%' IDENTIFIED BY '<password>';
GRANT ALL PRIVILEGES ON zammad_staging.* TO 'zammad_staging'@'%';
FLUSH PRIVILEGES;
```

## Insiden 8 — Asset belum pernah di-precompile → HTTP 500 di semua halaman
```
The asset "application.css" is not present in the asset pipeline.
```
**Sebab akar:** `git clone` TIDAK menyertakan hasil compile CSS/JS (beda dengan instalasi
produksi 3.4.0 lama yang kemungkinan dibuild dari tarball rilis resmi yang sudah include asset
ter-compile). `public/assets` cuma berisi file statis bawaan (icon/gambar/suara), bukan hasil
compile Sprockets.

**Fix (langsung ke container jalan):**
```bash
docker compose exec zammad-app env RAILS_ENV=production bundle exec rake assets:precompile
docker compose restart zammad-app
```
**Fix permanen (Dockerfile, untuk hop berikutnya):** tambahkan
`bundle exec rake assets:precompile RAILS_ENV=production` sebagai RUN step setelah
`bundle install` — sudah dimasukkan ke `Dockerfile` di folder ini.

## Insiden 9 — Nama rake task search index rebuild berbeda dari dokumentasi resmi terbaru
Dokumentasi resmi (versi terbaru) pakai `zammad:searchindex:rebuild`, tapi di 4.0.0 namespace-nya
`searchindex` langsung (tanpa prefix `zammad:`).
**Fix:** cek dulu dengan `bundle exec rake --tasks | grep -i -E "index|search"` sebelum asumsi
nama task — source-nya ada di `lib/tasks/search_index_es.rake` (task: `drop`, `create`,
`create_pipeline`, `reload`, `refresh`, `rebuild`, semua di namespace `searchindex:`).

## Insiden 10 — Disk host mendekati penuh (94-95%) → Elasticsearch mengunci index jadi read-only
```
cluster_block_exception: blocked by: [FORBIDDEN/12/index read-only / allow delete (api)]
```
ES otomatis mengunci index (block `read_only_allow_delete`) begitu node melewati *flood-stage
watermark* (default 95% disk usage) — proteksi bawaan supaya tidak ada write baru saat disk
kritis. **Detail penting:** block ini baru terlepas otomatis jika disk usage turun di bawah
*high watermark* (90%, BUKAN 95%) — jadi jika disk masih di 94%, block akan terus muncul lagi
walau sudah dibuka manual.

**Investigasi:** `df -h` → root partition (`/dev/vda3`, 130G) sudah terisi backup SQL 3.8GB
(`zammad_production_backup.sql`, dikompres jadi ~800MB pakai `gzip`), plus banyak image Docker
lama (`docker builder prune -f`, `docker image prune -f` membantu tapi tidak banyak), plus
server ini juga dipakai untuk banyak proyek lain di luar Zammad.

**Fix permanen:** bebaskan disk sampai jelas di bawah watermark (idealnya <85%).

**Stopgap jika disk belum bisa dibereskan cepat** (dipakai untuk lanjutkan testing hop ini):
```bash
# longgarkan watermark sementara
docker compose exec zammad-elasticsearch curl -X PUT "http://localhost:9200/_cluster/settings" \
  -H 'Content-Type: application/json' \
  -d '{"transient": {"cluster.routing.allocation.disk.watermark.low": "97%", "cluster.routing.allocation.disk.watermark.high": "98%", "cluster.routing.allocation.disk.watermark.flood_stage": "99%"}}'

# buka index yang sudah kadung ke-block
docker compose exec zammad-elasticsearch curl -X PUT "http://localhost:9200/_all/_settings" \
  -H 'Content-Type: application/json' -d '{"index.blocks.read_only_allow_delete": null}'
```
⚠️ **Wajib dikembalikan ke default** (`low: 85%`, `high: 90%`, `flood_stage: 95%`) begitu disk
benar-benar sudah dibereskan — jangan dibiarkan longgar permanen, karena disk yang terus-menerus
mendekati penuh berisiko ke MariaDB/Docker juga, bukan cuma ES.

**Data durasi reindex penuh** (161.884 tiket, 72rb+ user, 1 juta+ ticket_article — nested di
dalam dokumen ticket, bukan index terpisah): total sekitar **4,6 jam**, didominasi step
`reload Ticket` sendirian (**15144 detik / ~4,2 jam**) dan `reload User` (1296 detik / ~21,6
menit). Model lain semua di bawah 3 menit. **Ini jadi angka acuan penting untuk maintenance
window jika hop ini nanti dieksekusi ke produksi sungguhan** — reindex ES adalah tahap paling
lama dari seluruh proses upgrade, jauh melebihi migrasi schema database (1m33s).

## Konfigurasi nginx (domain diarahkan ke staging selama proses upgrade)

File: `/etc/nginx/conf.d/helpdesk.satu.solutions.conf` (backup produksi disimpan sebagai
`.conf.bak-produksi`). Port websocket **harus** `16043` (staging), bukan `16042` (produksi) —
ini bug yang sempat kejadian di draft awal, sudah dikoreksi.

## Isu minor yang belum tuntas (non-blocking)

Console browser menunjukkan `App.route dashboard:(error) | No permission for *` — kemungkinan
terkait permission baru dari migrasi `AddMissingPermissions`/`AgentCustomerPermission` yang belum
otomatis ter-assign ke role lama. Dashboard tetap render sempurna. **Belum diinvestigasi lebih
lanjut** — cek di Admin Panel → Roles jika ada fitur baru 4.0 yang terasa hilang.

## Insiden operasional selama proses ini

Server (`Koi-Server-Dev`) ternyata **shared** dengan banyak service lain di luar Zammad (OCR
frontend/middleware/mariadb, Headscale, Falco, dll) — total RAM 7.5GB dibagi banyak pihak.
Menjalankan produksi + staging Zammad bersamaan (2x Elasticsearch + 2x Rails) sempat membuat
produksi (`helpdesk.satu.solutions`) error 500 karena tekanan memori. **Keputusan yang diambil:**
matikan produksi (`zammad-audit`) selama seluruh proses upgrade berlangsung, dan arahkan domain
produksi ke staging lewat nginx — supaya tidak perlu jalankan 2 stack bersamaan sampai hop 7
selesai.
