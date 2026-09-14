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
| 7.0 (rilis Maret 2026) | 3.4.8 | ≥7.8, <10 (ES7 mulai deprecated) | **MySQL/MariaDB dihapus total** — PostgreSQL satu-satunya opsi | Rebuild search index wajib (perubahan ASCII-folding). Repo paket berganti skema baru (`dl.packager.io` → `go.packager.io`) |
| 7.1.3 (latest) | 3.4.9 | ≥7.8, <10 | PostgreSQL ≥13 | Redis ≥6 wajib |

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
