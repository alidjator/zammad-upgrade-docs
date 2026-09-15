# Roadmap Requirement Per-Hop

Sumber: docs.zammad.org (prerequisites/software.html, migrate-to-postgresql.html,
host-upgrade-repo-migration.html), rilis resmi zammad.com/en/product/releases/*,
serta `.ruby-version` & `Gemfile` di tiap tag GitHub `zammad/zammad`.

| Zammad | Ruby | Elasticsearch | Database | Catatan wajib |
|---|---|---|---|---|
| 3.4.0 (awal) | 2.6.5 | ≥5.5, ≤7.9 | MariaDB (bebas versi lama) | — |
| 4.0 | 2.6.6 | ≥6.5, ≤7.12 | — | Rails 5.2.4.5 |
| 5.0 | 2.7.4 | **≥7.8, <8** ⚠️ wajib upgrade ES sebelum hop ini | Postgres ≥9.3 / MySQL ≥5.5.8 | Node.js wajib untuk `assets:precompile` |
| 6.0 | 3.1.3 | ≥7.8, <9 | Pengumuman resmi: MySQL akan di-drop mulai 7.0 | **Redis jadi hard dependency.** Reverse-proxy wajib dikonfigurasi ulang untuk WebSocket (`/cable`) |
| — migrasi DB — | — | — | **MariaDB → PostgreSQL wajib selesai di sini** (tool migrasi `rake zammad:db:pgloader` baru ada mulai Zammad 5.3) | Lihat panduan resmi: migrate-to-postgresql.html |
| 7.0 (rilis Maret 2026) | 3.4.8 | ≥7.8, <10 (ES7 mulai deprecated) | **MySQL/MariaDB dihapus total** — PostgreSQL satu-satunya opsi | **Redis ≥6 wajib saat boot** (ditemukan lewat crash loop nyata di hop 6.0→7.0, BUKAN dari `.ruby-version`/dokumentasi resmi — lihat [hop-6.0-to-7.0/NOTES.md](hop-6.0-to-7.0/NOTES.md) Insiden 5). Rebuild search index wajib (perubahan ASCII-folding). Repo paket berganti skema baru (`dl.packager.io` → `go.packager.io`) |
| 7.1.3 (latest) | 3.4.9 | ≥7.8, <10 | PostgreSQL ≥13 | Redis ≥6 (sudah wajib sejak 7.0, lihat baris di atas) |

## Ringkasan perubahan besar 3.4.0 → 7.1.3 (agregat lintas-hop)

Tabel di atas fokus ke *requirement infrastruktur*. Ini agregat *perubahan
fitur/skema* terbesar per hop — bukan daftar lengkap (tiap `CHANGELOG.md` per hop
punya detail penuh), cuma highlight untuk gambaran cepat "apa yang berubah total":

- **3.4.0 → 4.0**: Rails 5.2.4.5. Fondasi awal, belum ada perubahan skema besar.
- **4.0 → 5.0**: pindah database ke MariaDB 10.11 (isu kompatibilitas versi-gap dengan
  Rails 6.0's mysql2 adapter — lihat [hop-4.0-to-5.0/CHANGELOG.md](hop-4.0-to-5.0/CHANGELOG.md)).
- **5.0 → 6.0**: Rails 6.1, adopsi Vite (build tool JS baru, gantikan Sprockets murni),
  **Redis jadi hard dependency**, WebSocket/ActionCable butuh config nginx baru — lihat
  [hop-5.0-to-6.0/CHANGELOG.md](hop-5.0-to-6.0/CHANGELOG.md).
- **Migrasi database**: MariaDB → PostgreSQL (11,3 juta baris, 0 error) — lihat
  [postgres-migration/NOTES.md](postgres-migration/NOTES.md).
- **6.0 → 7.0**: Rails 8.0, **MySQL/MariaDB dihapus total**, Yarn→pnpm, Node.js ≥20,
  **Redis ≥6 wajib**, 151 migrasi termasuk fitur AI Assistance, penghapusan Twitter/Slack,
  perubahan skema ASCII-folding search index — lihat
  [hop-6.0-to-7.0/CHANGELOG.md](hop-6.0-to-7.0/CHANGELOG.md) (perubahan terbesar di
  seluruh proyek).
- **7.0 → 7.1.3**: Node.js ≥24, 20 migrasi minor (AI Analytics, notifikasi standalone) —
  lihat [hop-7.0-to-7.1.3/CHANGELOG.md](hop-7.0-to-7.1.3/CHANGELOG.md) (hop paling
  ringan, tidak ada perubahan skema/index besar).

**Benang merah terbesar**: proyek ini pada dasarnya adalah 2 migrasi besar (database
MariaDB→PostgreSQL, dan build tool Sprockets→Vite/Yarn→pnpm) dibungkus di antara 4
lompatan major version Ruby/Rails standar.

## Titik kritis

1. **Sebelum hop ke 5.0**: Elasticsearch harus dinaikkan ke ≥7.8 (versi saat ini `6.8.23` sudah
   tidak cukup).
2. **Sebelum hop ke 7.0**: migrasi database MariaDB → PostgreSQL wajib selesai (dilakukan saat
   sudah di versi ≥5.3, sebelum menyentuh 6.0→7.0).
3. **Zammad 6.0 → 7.0**: reverse proxy nginx perlu penyesuaian config untuk WebSocket/ActionCable.

## Masalah yang berulang tiap hop (build from source)

Karena kita selalu `git clone` source code resmi per tag (bukan pakai image resmi/tarball rilis),
beberapa masalah generik cenderung muncul lagi di tiap hop — cek dulu sebelum build:

- **Gem yang di-yank dari rubygems.org** (seperti `mimemagic 0.3.5`) — perlu dicek per-versi apakah
  masih ada gem lama yang sudah tidak bisa diinstall, lalu patch `Gemfile.lock` manual.
- **Gem git-sourced dengan commit yang sudah hilang** (seperti `tcr` dari `zammad-deps/tcr`) — cek
  apakah masih bisa di-fetch sebelum build; kalau gem itu cuma dipakai untuk testing (grup `test`),
  aman dihapus dari `Gemfile` karena kita build dengan `--without development test`.
- **Asset belum pernah di-precompile** — git clone TIDAK menyertakan hasil compile CSS/JS.
  Selalu tambahkan `bundle exec rake assets:precompile RAILS_ENV=production` di Dockerfile.
- **Nama rake task search index berubah-ubah antar versi** — cek dulu dengan
  `bundle exec rake --tasks | grep -i -E "index|search"` sebelum asumsi nama task.
- **Build 3 image terpisah untuk zammad-app/websocket/scheduler** — ternyata sudah
  terjadi **sejak hop 3.4.0→4.0** (dikonfirmasi lewat penelusuran `journalctl -u
  docker`, bukan cuma ditemukan pertama kali di hop 6.0→7.0 seperti dugaan awal —
  lihat [DOWNTIME_ESTIMATE.md](DOWNTIME_ESTIMATE.md)). Kalau `docker-compose.yml`
  tidak diberi `image:` yang SAMA untuk ketiga service itu (Dockerfile-nya identik),
  Compose akan build 3x terpisah alih-alih sekali — bisa melipatgandakan waktu build
  sampai ~4-8x lebih lama dari seharusnya (hop 4.0→5.0 makan ~1 jam 2 menit karena ini).
  Selalu cek `docker-compose.yml` hop baru sudah pakai pola `image:` bersama sebelum
  build (lihat [hop-6.0-to-7.0/docker-compose.yml](hop-6.0-to-7.0/docker-compose.yml)
  sebagai contoh).

## Pelajaran operasional lintas-hop (bukan cuma build-from-source)

Pola di atas semuanya soal build dari source. Ada pola berulang lain yang sifatnya
lebih operasional/runtime, ditemukan di beberapa hop terpisah tapi baru disatukan di
sini — cek semuanya sebelum eksekusi hop manapun berikutnya:

- **Index Elasticsearch stale/nyangkut memblokir `searchindex:rebuild`** — terjadi
  berulang di hop 4.0→5.0, 5.0→6.0, dan 6.0→7.0 (Insiden 8), masing-masing dengan
  gejala sama: task rebuild gagal dengan `resource_already_exists_exception` padahal
  tahap drop index sebelumnya melaporkan sukses (kemungkinan race condition dengan
  background job, atau index lazy-created oleh model baru yang tidak ikut ter-drop).
  **Fix standar**: cek `_cat/indices` sebelum rebuild, hapus manual index
  `zammad_production_*` yang seharusnya sudah tidak ada (JANGAN hapus
  `.geoip_databases` — itu index sistem ES), baru rebuild dari kondisi bersih.
- **Kejutan versi runtime dependency yang tidak terdeteksi dari riset dokumentasi
  resmi/`.ruby-version`** — Redis jadi hard dependency (hop 5.0→6.0) dan lonjakan
  requirement ke Redis ≥6 (hop 6.0→7.0) sama-sama baru ketahuan lewat **crash loop
  nyata saat boot**, bukan dari riset `.ruby-version`/`Gemfile.lock` di awal.
  **Pelajaran**: riset requirement dari file dependency itu perlu, tapi tidak cukup —
  selalu siap diagnosis cepat kalau container crash-loop pasca-boot dengan pesan error
  yang jelas menyebut versi service pendukung (Redis, DB, dst.), jangan asumsikan
  riset awal sudah menangkap semua requirement.
- **"Container Up ≠ sehat"** — pelajaran eksplisit dari Insiden 7 hop 6.0→7.0:
  `assets:precompile` bisa gagal diam-diam saat boot (misal karena race dengan crash
  loop dependency lain), tapi `rails server` tetap bisa berjalan independen sehingga
  container berstatus "Up" — padahal SEMUA halaman menampilkan 500. **Selalu
  verifikasi `public/assets/` benar-benar berisi `application-*.css` dan
  `manifest.json`** setelah container pertama kali `Up`, jangan asumsikan sukses
  cuma dari status container atau `docker compose ps`.
- **Krisis disk berulang di hampir setiap hop** — sudah dicatat sebagai pola berulang
  sejak lama, tapi baru dikonfirmasi lewat `journalctl` bahwa akar masalah "build 3
  image terpisah" (poin di atas) sudah ada sejak hop pertama. Selalu cek
  `df -h` / `docker system df` sebelum DAN selama proses panjang (build, reindex) —
  jangan tunggu sampai kritis untuk mulai membersihkan.
