# Hop 4.0 → 5.0 — Catatan Lengkap (Status: ✅ Selesai & Tervalidasi)

## Ringkasan

Hop ini jauh lebih rumit dari 3.4.0→4.0 karena **dua masalah besar tumpang tindih**:
gem/Rails/Ruby yang perlu di-upgrade (relatif mulus), dan **inkompatibilitas MariaDB 11.8.3
(terlalu baru) dengan Rails 6.0's mysql2 adapter (terlalu lama)** yang butuh perubahan
strategi besar di tengah proses.

## Perubahan versi

- Ruby: 2.6.6 → **2.7.4**, base image pindah dari Debian **Stretch → Buster** (tidak ada
  tag `ruby:2.7.4-stretch`)
- Bundler: 1.17.3 → **2.2.20** (major version, bukan cuma minor)
- Rails: 5.2.4.5 → **6.0.4.1**
- Elasticsearch: 6.8.23 → **7.17.28** (wajib, ES lama tidak didukung Zammad 5.0)
- Database: **pindah dari MariaDB 11.8.3 (host) ke MariaDB 10.11 (container terpisah)** —
  lihat bagian "Keputusan besar" di bawah

## Kabar baik: gem tidak ada masalah

Berbeda dari hop sebelumnya, **`mimemagic` dan `tcr` sudah tidak jadi masalah** — Zammad
sendiri sudah memperbaikinya di Gemfile.lock 5.0.0 (mimemagic dihapus total dari
dependency tree, tcr diambil dari rubygems.org resmi bukan git fork). `bundle install`
langsung sukses tanpa patch apa pun.

## Masalah yang ditemukan & fix-nya (urutan kejadian)

### 1. `config/database.yml` tidak ada saat build (assets:precompile gagal)
```
Could not load database configuration. No such file - ["config/database.yml"]
```
**Sebab:** `database.yml` di-mount lewat volume Docker Compose saat container RUNTIME,
tapi belum ada saat `docker build`. Rails 6.0 (beda dari 5.2) memvalidasi config database
lebih ketat saat boot — bahkan untuk `assets:precompile`.

**Fix permanen: pindahkan `assets:precompile` dari Dockerfile (build time) ke `command:`
di docker-compose.yml (runtime)** — dijalankan tiap kali container start, setelah
`database.yml` asli ter-mount. Konsekuensi: startup jadi lebih lambat (perlu re-precompile
tiap restart), tapi ini satu-satunya cara robust karena DB baru bisa diakses saat runtime.

### 2. MariaDB 11.8.3 terlalu baru untuk Rails 6.0's mysql2 adapter — 3 bug berbeda

**Bug 2a:** `mysql_variable('version').split('-')` gagal karena `@@version` ke-parse
sebagai Integer, bukan String.

**Bug 2b:** `create_time_zone_conversion_attribute?(name, cast_type)` dipanggil dengan
`name` berupa BigDecimal, bukan nama atribut — cuma muncul kalau ada model yang schema-nya
di-load (`inherited` hook ActiveRecord).

**Bug 2c:** Setelah bug 2b ditambal, muncul lagi: `ArgumentError: invalid value for
BigDecimal(): "2026-09-12 02:59:29.683"` — timestamp datetime dipaksa jadi BigDecimal.

**Akar masalah:** sejak MariaDB 10.2.7, literal di `COLUMN_DEFAULT`
(`information_schema.COLUMNS`) mulai diberi tanda kutip untuk membedakan dari ekspresi.
Rails 6.0's parser metadata kolom (era 2019-2021) tidak didesain untuk format ini, dan
MariaDB 11.x kemungkinan mengubah lebih jauh detail protokol type-reporting, menyebabkan
tipe kolom `decimal` dan `datetime` tertukar/salah baca untuk kolom-kolom tertentu.

**Ini BUKAN bug tunggal yang bisa ditambal sekali jalan** — 3 gejala berbeda muncul cuma
untuk sampai tahap boot, belum migrate/reindex. Pola whack-a-mole ini kemungkinan berulang
di hop 6.0 (masih Rails 6.0.x/6.1.x).

## Keputusan besar: pindah ke MariaDB 10.11 khusus staging

Alih-alih terus menambal bug demi bug di MariaDB 11.8.3, **staging hop 4.0→6.0 sekarang
pakai container MariaDB 10.11 terpisah** (seumuran rilis Zammad 5.0/6.0), bukan MariaDB
11.8.3 di host. Setelah dites tanpa patch sama sekali di MariaDB 10.11 (fresh clone Zammad
5.0.0, tanpa patch mysql2.rb maupun initializer BigDecimal), **build & boot langsung sukses
tanpa satu pun dari 3 bug di atas muncul** — mengonfirmasi akar masalahnya memang murni
versi MariaDB, bukan bug Zammad.

**Arsitektur baru:**
- Service `zammad-mariadb-legacy` (image `mariadb:10.11`) ditambahkan ke docker-compose.yml
- `database.yml` host diarahkan ke `zammad-mariadb-legacy` (nama service, bukan
  `host.docker.internal` lagi — koneksi jadi container-to-container dalam network compose
  yang sama, bukan container-ke-host)
- Restore data: `mysqldump` dari MariaDB produksi (11.8.3) di-restore ke MariaDB 10.11 ini
  — restore SQL dump lama ke server lebih lama umumnya kompatibel tanpa masalah
- **Restore WAJIB pakai user `root`, bukan user aplikasi terbatas** — dump berisi definisi
  trigger dengan `DEFINER=`crontab`@`%`\`` yang butuh privilege SUPER

**Rencana selanjutnya (tidak berubah dari sebelumnya):** migrasi ke PostgreSQL tetap
dilakukan nanti, di titik ketika Zammad sudah mencapai versi ≥5.3 (pakai tool resmi
`rake zammad:db:pgloader`), sebelum hop ke 7.0 — sekarang sumbernya MariaDB 10.11 ini,
bukan lagi MariaDB 11.8.3.

## Insiden disk penuh (2x, lebih parah dari hop sebelumnya)

Disk sempat mencapai **100% penuh (911MB tersisa)** di tengah proses — lebih parah dari
hop 1 (94-95%). Penyebab utama kali ini: **build cache Docker menumpuk sampai 11.43GB**
(dari berkali-kali `docker compose build` selama debugging panjang), plus volume ES lama
yang tidak terpakai (`zammad-staging_es-data` dari ES 6.8.23 hop sebelumnya = 4.06GB,
`zammad-audit_es-data` dari produksi yang sudah dimatikan = 1.6GB).

**Fix:**
```bash
docker builder prune -af          # -a penting: hapus SEMUA cache reclaimable, bukan cuma dangling
docker volume rm zammad-staging_es-data zammad-audit_es-data
```
Total terbebas: ~17GB (dari 100% ke 88%).

**Pelajaran untuk hop berikutnya:** jalankan `docker builder prune -af` secara rutin
setelah beberapa kali build gagal berturut-turut selama debugging — jangan tunggu sampai
disk kritis. Volume ES versi lama yang sudah tidak dipakai (setelah pindah ke image ES
baru) juga harus dihapus segera, bukan dibiarkan menumpuk.

## Rebuild search index — masalah index nyangkut dari percobaan gagal

Setelah disk dibereskan dan watermark dibuka lagi, retry `searchindex:rebuild` gagal lagi
dengan `resource_already_exists_exception` untuk index `user` — sisa index dari percobaan
sebelumnya yang gagal di tengah jalan (drop index di awal rebuild tidak selalu bersih kalau
proses sebelumnya terhenti paksa). **Fix:** hapus manual semua index Zammad by wildcard
sebelum retry:
```bash
docker compose exec zammad-elasticsearch curl -X DELETE "http://localhost:9200/localhost.localdomain_zammad_production_*"
```
(hati-hati jangan hapus `.geoip_databases`, itu index sistem bawaan ES, bukan punya Zammad)

**Durasi reindex penuh:** ~4 jam 10 menit total (`tickets` 13570s/~3,8 jam, `users`
1244s/~20,7 menit) — konsisten dengan hop sebelumnya, tetap jadi tahap paling lama.

## Nama rake task search index (masih sama seperti hop sebelumnya)

Namespace `searchindex:` (bukan `zammad:searchindex:`), tidak muncul di `rake --tasks`
karena tidak ada deskripsi — selalu cek `lib/tasks/search_index_es.rake` langsung kalau
ragu nama task berubah lagi di versi berikutnya.

## Verifikasi akhir

- ✅ UI menampilkan "This is Zammad version 5.0.x" di Admin → System → Version
- ✅ Search berfungsi (50 tiket ditemukan untuk kata kunci uji)
- ✅ Console browser cuma warning ringan yang sudah dikenal (CSP, notification permission)
