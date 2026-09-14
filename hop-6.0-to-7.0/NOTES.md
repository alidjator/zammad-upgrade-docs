# Hop 6.0 → 7.0 — Catatan (Status: 🔧 Uji `bundle install`/`pnpm install` tervalidasi, build image penuh belum)

## Requirement (dari riset `.ruby-version`, `Gemfile.lock`, `package.json` di tag `7.0.0`)

| Item | Versi | Perubahan dari hop 6.0 |
|---|---|---|
| Ruby | 3.4.8 | naik dari 3.1.3 |
| Bundler | 2.6.9 | naik dari 2.4.1 |
| Rails | 8.0.4 | **loncat dari 6.1.7.3 — Rails 7 dilewati total** |
| Node.js | ≥20 | naik dari ≥16 |
| Package manager JS | **pnpm ≥10** (`packageManager: "pnpm@10.29.1"`) | **ganti dari Yarn** |
| Database | PostgreSQL saja | mysql2 sudah hilang total dari Gemfile.lock — sesuai pengumuman resmi di hop 6.0 |
| Base OS image | Debian Bookworm (`ruby:3.4.8-bookworm`) | naik dari Buster — tidak ada tag `3.4.8-buster` di Docker Hub |
| Elasticsearch | ≥7.8, <10 (tidak berubah) | ES 7.17.28 tetap dipakai, TAPI wajib `searchindex:rebuild` (perubahan ASCII-folding) |

## Kabar baik: tidak ada friksi database

Karena staging sudah dipindah ke PostgreSQL (lihat [postgres-migration/NOTES.md](../postgres-migration/NOTES.md))
sebelum hop ini dimulai, hop 6.0→7.0 **tidak perlu langkah migrasi data tambahan** —
cukup pastikan `database.yml` tetap mengarah ke PostgreSQL host yang sama.
`zammad-mariadb-legacy` dibiarkan tetap ada di compose (rollback safety net) tapi
sudah tidak ada satupun service yang `depends_on` ke sana mulai hop ini.

## Perubahan toolchain yang belum pernah dihadapi sebelumnya

- **pnpm menggantikan Yarn** — Zammad mem-pin versi pnpm lewat field `packageManager`
  di `package.json`. Solusi: pakai `corepack enable` (bawaan Node ≥16.9, aktif di
  Node 20 image NodeSource) supaya versi pnpm yang benar otomatis ter-fetch saat
  `pnpm install` pertama kali dipanggil — tidak perlu `npm install -g pnpm@<versi>` manual.
- **Debian Buster → Bookworm** — perlu dicek ulang apakah masalah EOL-archive
  (`archive.debian.org`) yang selalu muncul di image Buster (hop 1-3) juga terjadi di
  Bookworm. Bookworm rilis 2023 dan kemungkinan masih dalam masa dukungan resmi saat
  hop ini dieksekusi (2026) — coba build TANPA workaround dulu, baru tambahkan kalau
  `apt-get update` gagal.
- **`tcr`/`vcr` gem** — masih ada di Gemfile sebagai `require: false`, tanpa git source
  (beda dari kasus hop 1 yang git-sourced dan gem-nya sudah mati). Tidak perlu tindakan.

## Yang TIDAK berubah dari hop sebelumnya

- Pola `assets:precompile` tetap dijalankan di runtime (`command:`), bukan build time —
  alasan yang sama sejak hop 3 (initializer butuh koneksi DB live).
- `REDIS_URL`, `BACKGROUND_SERVICES_LOG_TO_STDOUT`, `script/background-worker.rb start`
  tetap sama seperti hop 6.0.

## Poin dari ROADMAP.md yang perlu diverifikasi saat eksekusi

- **Rebuild search index wajib** karena perubahan ASCII-folding — jangan lupa jalankan
  `searchindex:rebuild` setelah migrasi selesai (beda dari migrasi Postgres kemarin yang
  TIDAK butuh rebuild index).
- Nama rake task search index perlu dicek ulang lagi (`bundle exec rake --tasks | grep -i
  -E "index|search"`) — sudah berubah nama 2x di hop-hop sebelumnya.
- Repo paket resmi Zammad pindah skema `dl.packager.io` → `go.packager.io` — **tidak
  relevan untuk kita** karena kita build dari source (`git clone`), bukan install lewat
  repo paket resmi.

## Verifikasi `bundle install` + `pnpm install` di container sementara (tervalidasi)

Dijalankan di `ruby:3.4.8-bookworm` sebelum menulis Dockerfile final — pola yang sama
dipakai di setiap hop untuk menangkap masalah gem/paket lebih awal.

- **`apt-get update` di Debian Bookworm berjalan mulus, TIDAK perlu workaround
  `archive.debian.org`** — beda dari semua image Buster (hop 1-3) yang selalu butuh
  redirect ke archive EOL. Bookworm masih dalam masa dukungan resmi Debian saat hop
  ini dieksekusi (2026).
- **`pnpm install --frozen-lockfile` sukses 100% di percobaan pertama** — `corepack
  enable` berhasil auto-fetch pnpm 10.29.1 sesuai pin `packageManager` di
  `package.json`, tidak perlu install manual versi tertentu.
- **Bug ditemukan: gem `rszr` (image resizing, dependency Zammad) gagal build native
  extension** — `checking for pkg-config for imlib2... not found`. Root cause: paket
  `pkg-config` sendiri tidak ter-install (Debian tidak menyertakannya secara default),
  jadi meskipun `libimlib2-dev` sudah ada di rencana Dockerfile, extconf tidak bisa
  mendeteksinya tanpa binary `pkg-config`. **Fix:** tambahkan paket `pkg-config` ke
  daftar `apt-get install` di [Dockerfile](Dockerfile) (sebelum `libimlib2-dev`).
  Setelah fix, `bundle install --without development test` sukses penuh: **128
  Gemfile dependencies, 252 gems terinstall, `Bundle complete!`**.
- Peringatan `platform specific gems ... Please run bundle lock
  --normalize-platforms` muncul (untuk `pg`, `ffi`, `nokogiri`) — ini cuma saran
  housekeeping dari Bundler, bukan error, aman diabaikan untuk staging.

## Hasil validasi

- ✅ `pnpm install --frozen-lockfile` — 0 error
- ✅ `bundle install --without development test` — 0 error (setelah fix `pkg-config`)
- ⬜ Build image Docker penuh — belum dilakukan
- ⬜ Migrasi `db:migrate` — belum dilakukan
- ⬜ `searchindex:rebuild` — belum dilakukan
- ⬜ Verifikasi UI/search — belum dilakukan

<!-- Lanjutkan bagian ini dengan hasil eksekusi nyata berikutnya: error build image,
migrasi, reindex, dan hasil validasi UI/search setelah hop ini benar-benar selesai. -->
