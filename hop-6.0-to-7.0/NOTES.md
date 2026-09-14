# Hop 6.0 → 7.0 — Catatan (Status: 🔜 Riset selesai, eksekusi belum dimulai)

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

## Rencana verifikasi sebelum build penuh

Sebelum menulis Dockerfile final, uji dulu `bundle install` dan `pnpm install` di
container sementara berbasis `ruby:3.4.8-bookworm` — pola yang sama yang selalu
dipakai di setiap hop untuk menangkap masalah gem/paket lebih awal tanpa menunggu
build image penuh.

<!-- Lanjutkan bagian ini dengan hasil eksekusi nyata: error yang ditemukan, fix yang
dipakai, hasil validasi UI/search setelah hop ini benar-benar dijalankan di server. -->
