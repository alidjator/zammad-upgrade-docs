# Hop 7.0 → 7.1.3 — Catatan (Status: 🔧 Uji bundle/pnpm install tervalidasi, build image belum)

## Verifikasi `bundle install` + `pnpm install` di container sementara (tervalidasi)

Dijalankan di `ruby:3.4.9-bookworm` — **sukses penuh di percobaan pertama, tanpa
insiden baru**, karena semua pelajaran dari hop 6.0→7.0 (pkg-config, `CI=true`) sudah
dimasukkan proaktif ke Dockerfile template sebelum eksekusi:
- `apt-get update` di Bookworm tetap mulus (sama seperti hop sebelumnya)
- `pnpm install` — Node.js 24.21.0, pnpm 10.33.3 (auto via corepack), 0 error
- `bundle install --without development test` — **`Bundle complete! 127 Gemfile
  dependencies, 250 gems now installed.`**, 0 error (gem `rszr` yang dulu bermasalah
  di hop 6.0→7.0 kali ini langsung sukses karena `pkg-config` sudah ada dari awal)

## Ringkasan requirement (hasil riset di tag `7.1.3` GitHub)

| Item | Versi | Perubahan dari hop 6.0→7.0 |
|---|---|---|
| Ruby | 3.4.9 | naik dari 3.4.8 (patch saja) |
| Bundler | 2.6.9 | tidak berubah |
| Rails | 8.0.5.1 | naik dari 8.0.4 (patch saja) |
| Node.js | **≥24** | naik dari ≥20 |
| pnpm | ≥10, dipin `pnpm@10.33.3` | naik dari `10.29.1` (otomatis lewat corepack) |
| Database | PostgreSQL saja | tidak berubah |
| Base OS image | Debian Bookworm (`ruby:3.4.9-bookworm`) | tidak berubah — tag Docker Hub sudah dikonfirmasi ada |
| Elasticsearch | ≥7.8, <10 (tidak berubah versi) | tetap 7.17.28 |

Ini hop **paling ringan** dari seluruh proyek — cuma naik versi patch untuk
Ruby/Rails, dan satu bump minor untuk Node.js. Tidak ada perubahan database, tidak ada
perubahan base OS, tidak ada gem git-sourced baru yang bermasalah (tcr/vcr masih sama
seperti sebelumnya, tanpa git source).

## Temuan dari `BREAKING_CHANGES.md` Zammad 7.1

- **Elasticsearch 7 resmi dinyatakan deprecated** — bukan requirement keras di 7.1.3
  (masih ≥7.8,<10, ES 7.17.28 kita tetap valid), tapi Zammad mengumumkan versi
  SETELAH 7.1.3 akan mewajibkan ES 8+. Tidak relevan untuk hop ini (7.1.3 adalah
  rilis terakhir di roadmap proyek), dicatat untuk referensi kalau proyek dilanjutkan.
- **Fulltext search asciifolding** — sudah diaktifkan sejak 7.0 (bukan baru di 7.1),
  sudah kita tangani saat hop sebelumnya.
- **nginx wajib `proxy_http_version 1.1;` di `location /`** (berlaku sejak 7.0) —
  **dicek ke config nginx production, ternyata SUDAH ADA** (kemungkinan ditambahkan
  saat penyesuaian WebSocket di hop 5.0→6.0). Tidak perlu tindakan apa pun.
- Perubahan lain (Calendar iCal wajib URL, rename `Exceptions::UnprocessableEntity`)
  tidak relevan — instance ini tidak memakai fitur calendar iCal file lokal, dan tidak
  ada integrasi custom yang memanggil exception class tersebut.

## Yang TIDAK berubah dari hop sebelumnya (best practice tetap berlaku)

- Pola `assets:precompile` tetap dijalankan di runtime (`command:`), bukan build time.
- `pkg-config` + `libimlib2-dev` tetap wajib di Dockerfile (gem `rszr`).
- `ENV CI=true` tetap wajib sebelum `pnpm install` (hindari `ERR_PNPM_ABORTED_REMOVE_MODULES_DIR_NO_TTY`).
- `redis:7-alpine` tetap dipakai (requirement Redis ≥6 sejak 7.0, tidak berubah).
- Strategi **1 image dipakai 3 service** (`image: zammad-staging-app:7.1.3` sama di
  app/websocket/scheduler) — WAJIB, ini bukan cuma optimisasi tapi pelajaran dari
  krisis disk berulang di SEMUA hop sebelumnya (bahkan hop 1 dan 2 ternyata sudah
  kena masalah build-3x ini, baru ketahuan lewat penelusuran journalctl).
- **Verifikasi `public/assets/` berisi `application-*.css` setelah container pertama
  kali `Up`** — jangan asumsikan sukses cuma dari status container (pelajaran Insiden 7
  hop 6.0→7.0: container bisa "Up" padahal asset pipeline gagal diam-diam).
- Cek `_cat/indices` bersih sebelum `searchindex:rebuild` — index stale dari percobaan
  gagal sebelumnya bisa memblokir rebuild (Insiden 8).
- Pantau disk selama reindex — ES bisa masuk mode `read_only_allow_delete` kalau disk
  lewat flood-stage watermark, dan block ini TIDAK otomatis lepas (Insiden 9).

## Metode pengukuran waktu (standar baru, lihat DOWNTIME_ESTIMATE.md)

Manfaatkan pelaporan bawaan tiap command + simpan ke file (bukan `time`, bukan cuma
andalkan buffer `screen`):
- Build: `docker compose build zammad-app 2>&1 | tee build-hop7.1.3.log` (Docker Buildx
  sudah cetak total durasi di baris ringkasannya sendiri)
- Migrate: tidak perlu redirect tambahan — `log/production.log` otomatis persisten,
  ambil durasi dari `grep "Migrating to" log/production.log` (baris pertama & terakhir)
- Reindex: `... rake zammad:searchindex:rebuild 2>&1 | tee reindex-hop7.1.3.log`
  ("done in X seconds" cuma ke STDOUT, wajib di-tee)

## Rencana verifikasi sebelum build penuh

Uji `bundle install` + `pnpm install` dulu di container sementara berbasis
`ruby:3.4.9-bookworm` sebelum build image penuh — pola yang selalu dipakai di setiap
hop untuk menangkap masalah gem/paket lebih awal.

<!-- Lanjutkan bagian ini dengan hasil eksekusi nyata setelah hop ini benar-benar
dijalankan di server. -->
