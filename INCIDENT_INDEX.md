# Indeks Insiden Lintas-Hop

Satu tabel ringkas untuk lookup cepat "pernah ketemu error ini sebelumnya?" — tanpa
harus membuka satu-satu 5 file `NOTES.md`. Detail penuh (log error, root cause, fix
lengkap) selalu ada di link masing-masing.

## Hop 3.4.0 → 4.0 (10 insiden)

| # | Ringkasan | Detail |
|---|---|---|
| 1 | `mysql`/`mysqldump` client tidak ditemukan di container MariaDB (rename ke `mariadb`) | [NOTES.md](hop-3.4.0-to-4.0/NOTES.md#insiden-1--mysqlmysqldump-client-tidak-ditemukan-di-container-mariadb) |
| 2 | Bash history expansion pada password mengandung `!` | [NOTES.md](hop-3.4.0-to-4.0/NOTES.md#insiden-2--bash-history-expansion-pada-password-mengandung-) |
| 3 | Password salah: pakai password `zammad_audit` untuk login `root` | [NOTES.md](hop-3.4.0-to-4.0/NOTES.md#insiden-3--password-salah-pakai-password-zammad) |
| 4 | `mimemagic (0.3.5)` sudah di-yank dari rubygems.org | [NOTES.md](hop-3.4.0-to-4.0/NOTES.md#insiden-4--mimemagic-035-sudah-di-yank-dari-rubygemsorg) |
| 5 | Gem `tcr` — git commit sudah hilang dari repo `zammad-deps/tcr` | [NOTES.md](hop-3.4.0-to-4.0/NOTES.md#insiden-5--gem-tcr--git-commit-sudah-hilang-dari-repo-zammad-depstcr) |
| 6 | Fix Gemfile dijalankan via container sementara (bukan langsung di Dockerfile) | [NOTES.md](hop-3.4.0-to-4.0/NOTES.md#insiden-6--proses-fix-di-atas-dijalankan-via-container-sementara-bukan-langsung-di-dockerfile) |
| 7 | MariaDB user hanya bisa login dari `localhost`, bukan dari IP container | [NOTES.md](hop-3.4.0-to-4.0/NOTES.md#insiden-7--mariadb-user-hanya-bisa-login-dari-localhost-bukan-dari-ip-container) |
| 8 | Asset belum pernah di-precompile → HTTP 500 di semua halaman | [NOTES.md](hop-3.4.0-to-4.0/NOTES.md#insiden-8--asset-belum-pernah-di-precompile--http-500-di-semua-halaman) |
| 9 | Nama rake task search index rebuild berbeda dari dokumentasi resmi terbaru | [NOTES.md](hop-3.4.0-to-4.0/NOTES.md#insiden-9--nama-rake-task-search-index-rebuild-berbeda-dari-dokumentasi-resmi-terbaru) |
| 10 | Disk host 94-95% → Elasticsearch mengunci index jadi read-only | [NOTES.md](hop-3.4.0-to-4.0/NOTES.md#insiden-10--disk-host-mendekati-penuh-94-95--elasticsearch-mengunci-index-jadi-read-only) |

## Hop 4.0 → 5.0 (4 insiden)

| # | Ringkasan | Detail |
|---|---|---|
| 1 | `config/database.yml` tidak ada saat build (`assets:precompile` gagal) | [NOTES.md](hop-4.0-to-5.0/NOTES.md#insiden-1--configdatabaseyml-tidak-ada-saat-build-assetsprecompile-gagal) |
| 2 | MariaDB 11.8.3 terlalu baru untuk Rails 6.0's mysql2 adapter — 3 bug berbeda | [NOTES.md](hop-4.0-to-5.0/NOTES.md#insiden-2--mariadb-1183-terlalu-baru-untuk-rails-60s-mysql2-adapter--3-bug-berbeda) |
| 3 | Disk penuh 2x, lebih parah dari hop sebelumnya (build cache 11,43GB) | [NOTES.md](hop-4.0-to-5.0/NOTES.md#insiden-3--disk-penuh-2x-lebih-parah-dari-hop-sebelumnya) |
| 4 | Rebuild search index gagal — index nyangkut dari percobaan sebelumnya | [NOTES.md](hop-4.0-to-5.0/NOTES.md#insiden-4--rebuild-search-index-masalah-index-nyangkut-dari-percobaan-gagal) |

## Hop 5.0 → 6.0 (7 insiden)

| # | Ringkasan | Detail |
|---|---|---|
| 1 | Lupa transfer file Dockerfile/docker-compose.yml baru ke server | [NOTES.md](hop-5.0-to-6.0/NOTES.md#insiden-1--lupa-transfer-file-dockerfiledocker-composeyml-baru-ke-server) |
| 2 | `script/scheduler.rb` di-rename jadi `script/background-worker.rb` | [NOTES.md](hop-5.0-to-6.0/NOTES.md#insiden-2--scriptschedulerrb-di-rename-jadi-scriptbackground-workerrb) |
| 3 | Redis jadi hard dependency — gagal connect ke `localhost:6379` | [NOTES.md](hop-5.0-to-6.0/NOTES.md#insiden-3--redis-jadi-hard-dependency--gagal-connect-ke-localhost6379) |
| 4 | Vite build gagal — `Errno::ENOENT: yarn` | [NOTES.md](hop-5.0-to-6.0/NOTES.md#insiden-4--vite-build-gagal--errnoenoent-yarn) |
| 5 | Nama rake task search index berubah LAGI (3 kali, 3 hop berturut-turut) | [NOTES.md](hop-5.0-to-6.0/NOTES.md#insiden-5--nama-rake-task-search-index-berubah-lagi-3-kali-berubah-3-hop-berturut-turut) |
| 6 | Disk penuh lagi (95% → cleanup 6,5GB build cache) | [NOTES.md](hop-5.0-to-6.0/NOTES.md#insiden-6--disk-penuh-lagi-95--butuh-cleanup-65gb-build-cache) |
| 7 | Index ES nyangkut lagi dari percobaan gagal (partial reload) | [NOTES.md](hop-5.0-to-6.0/NOTES.md#insiden-7--index-es-nyangkut-lagi-dari-percobaan-gagal-partial-reload) |

## Hop 6.0 → 7.0 (9 insiden — paling banyak di seluruh proyek)

| # | Ringkasan | Detail |
|---|---|---|
| 1 | `bundle install` gagal: gem `rszr` butuh `pkg-config` | [NOTES.md](hop-6.0-to-7.0/NOTES.md#insiden-1--bundle-install-gagal-gem-rszr-butuh-pkg-config) |
| 2 | Debian Bookworm TIDAK butuh workaround archive.debian.org | [NOTES.md](hop-6.0-to-7.0/NOTES.md#insiden-2--debian-bookworm-tidak-butuh-workaround-archivedebianorg) |
| 3 | `pnpm install` gagal di Docker build: butuh `CI=true` | [NOTES.md](hop-6.0-to-7.0/NOTES.md#insiden-3--pnpm-install-gagal-di-docker-build-butuh-citrue) |
| 4 | Krisis disk berulang selama build (paling parah di seluruh proyek) | [NOTES.md](hop-6.0-to-7.0/NOTES.md#insiden-4--krisis-disk-berulang-selama-build-paling-parah-di-hop-ini) |
| 5 | Redis 5 tidak lagi didukung (ditemukan lewat crash loop) | [NOTES.md](hop-6.0-to-7.0/NOTES.md#insiden-5--redis-5-tidak-lagi-didukung-ditemukan-lewat-crash-loop) |
| 6 | Bug urutan migrasi resmi Zammad: `recent_closes` belum ada saat dibutuhkan | [NOTES.md](hop-6.0-to-7.0/NOTES.md#insiden-6--bug-urutan-migrasi-resmi-zammad-recent) |
| 7 | Error 500 pasca-migrasi: asset pipeline tidak pernah ter-precompile | [NOTES.md](hop-6.0-to-7.0/NOTES.md#insiden-7--error-500-pasca-migrasi-asset-pipeline-tidak-pernah-ter-precompile) |
| 8 | Index Elasticsearch stale menghalangi `searchindex:rebuild` | [NOTES.md](hop-6.0-to-7.0/NOTES.md#insiden-8--index-elasticsearch-stale-menghalangi-searchindexrebuild) |
| 9 | Elasticsearch masuk mode `read_only_allow_delete` di tengah reload data | [NOTES.md](hop-6.0-to-7.0/NOTES.md#insiden-9--elasticsearch-masuk-mode-read) |

## Hop 7.0 → 7.1.3 (2 insiden — hop paling ringan)

| # | Ringkasan | Detail |
|---|---|---|
| 1 | Disk 100% penuh tepat saat `docker compose up -d` | [NOTES.md](hop-7.0-to-7.1.3/NOTES.md#insiden-1--disk-100-penuh-tepat-saat-docker-compose-up--d) |
| 2 | Error 500 transisional saat kode baru jalan sebelum migrasi (normal, bukan bug) | [NOTES.md](hop-7.0-to-7.1.3/NOTES.md#insiden-2--error-500-transisional-saat-kode-baru-jalan-sebelum-migrasi-normal-bukan-bug) |

## Pola yang berulang lintas-hop

Lihat [ROADMAP.md § "Pelajaran operasional lintas-hop"](ROADMAP.md#pelajaran-operasional-lintas-hop-bukan-cuma-build-from-source)
untuk pola yang muncul lebih dari sekali (index ES stale, kejutan versi dependency,
krisis disk, "Container Up ≠ sehat") — sudah disatukan di sana, tidak diulang di sini.
