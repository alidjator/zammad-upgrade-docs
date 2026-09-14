# Changelog — Hop 6.0 → 7.0

Sumber: CHANGELOG resmi Zammad (github.com/zammad/zammad, tag `7.0.0`) + riset
`Gemfile.lock`/`package.json` langsung di tag tersebut.

## Environment

- Ruby 3.1.3 → **3.4.8**
- Bundler 2.4.1 → **2.6.9**
- Rails 6.1.7.3 → **8.0.4** (Rails 7 dilewati total)
- Node.js ≥16 → **≥20**
- Yarn → **pnpm ≥10** (dipin ke `pnpm@10.29.1` lewat field `packageManager`)
- Base image Docker: Debian Buster → **Debian Bookworm** (tidak ada lagi tag Ruby
  untuk Buster)

## Database

- **MySQL/MariaDB dihapus total** — gem `mysql2` sudah tidak ada di Gemfile.lock.
  PostgreSQL menjadi satu-satunya adapter didukung.

## Distribusi / packaging

- Repo paket resmi berpindah skema dari `dl.packager.io` ke `go.packager.io` — tidak
  berdampak ke kita karena build dari source, bukan lewat repo paket OS.

## Redis

- **Redis ≥6 wajib saat boot** — ditemukan lewat crash loop nyata di staging
  (`Error: incompatible Redis version (6+ required; 5.0.14 found)`), bukan dari riset
  dokumentasi resmi/`.ruby-version` di awal (ROADMAP.md sebelumnya mencatat requirement
  ini cuma di baris 7.1.3). Berlaku juga untuk task `assets:precompile` karena task itu
  turut mem-boot environment Rails penuh. Lihat [NOTES.md](NOTES.md) Insiden 5 & 7.

## Skema database — perubahan penting yang ditemukan saat eksekusi

- **Tabel baru `recent_closes`** (migrasi `20251106095318`) — mendukung fitur pelacakan
  "recently closed" per user/objek. **Perhatian urutan migrasi:** migrasi
  `20241106073757 TaskbarAddUniquenessIndex` (November 2024, jauh lebih awal) sudah
  memicu callback yang butuh tabel ini lewat kode model 7.0.0 — migrasi
  `CreateRecentCloses` harus dijalankan LEBIH DULU secara manual (`db:migrate:up
  VERSION=20251106095318`) sebelum `db:migrate` normal, kalau tidak migrasi akan gagal
  dengan `PG::UndefinedTable`. Ini bug urutan/desain di source Zammad sendiri, muncul
  karena kita menjalankan puluhan migrasi historis sekaligus. Lihat
  [NOTES.md](NOTES.md) Insiden 6 untuk analisis lengkap.
- Total 78 migrasi berjalan dari `TaskbarAddUniquenessIndex` (Nov 2024) sampai
  `Pr5952FixTypos` (Feb 2026) — mencakup fitur AI Assistance (text tools, ticket
  summarize, AI agents), penghapusan integrasi Twitter & Slack, checklist, webhook
  bearer token, dan banyak penyesuaian permission/UI kecil.

## Search / Elasticsearch

- Requirement versi tidak berubah (≥7.8, <10, tetap 7.17.28) — tapi **skema
  ASCII-folding index berubah**, mewajibkan `searchindex:rebuild` penuh setelah
  upgrade meski versi ES-nya sendiri tidak naik.
- Rebuild otomatis (task rake) melakukan drop index lama, tapi bisa meninggalkan index
  stale kalau ada race condition dengan background job — verifikasi `_cat/indices`
  bersih sebelum rebuild kalau task gagal dengan `resource_already_exists_exception`.
  Lihat [NOTES.md](NOTES.md) Insiden 8.

<!-- Tambahkan di sini kalau ada perubahan skema/tabel/kolom/fitur lain yang ditemukan
selama eksekusi nyata (migrasi Rails, fitur baru/dihapus di UI, dsb). -->
