# Hop 6.0 → 7.0 — Changelog

Format mengikuti [Keep a Changelog](https://keepachangelog.com/) (kategori
Added/Changed/Removed), diadaptasi untuk konteks upgrade infrastruktur — bukan rilis
versi aplikasi sendiri.

Sumber: CHANGELOG resmi Zammad (github.com/zammad/zammad, tag `7.0.0`) + riset
`Gemfile.lock`/`package.json` langsung di tag tersebut.

## Added

- Tabel `recent_closes` (migrasi `20251106095318`) — mendukung fitur pelacakan
  "recently closed" per user/objek.
- Fitur AI Assistance (text tools, ticket summarize, AI agents), checklist, webhook
  bearer token — bagian dari **151 migrasi total** yang berjalan (dikonfirmasi lewat
  `grep "Migrating to" log/production.log`, bukan 78 seperti dugaan awal — angka 78
  keliru karena cuma menghitung sisa migrasi "down" SETELAH percobaan pertama sempat
  berhasil menjalankan puluhan migrasi lama 2022-2024 sebelum gagal di
  `recent_closes`), dari `SettingAddStoreProviderS3` (Sept 2022) sampai
  `Pr5952FixTypos` (Feb 2026).

## Changed

- Ruby 3.1.3 → **3.4.8**, Bundler 2.4.1 → **2.6.9**, Rails 6.1.7.3 → **8.0.4** (Rails
  7 dilewati total), Node.js ≥16 → **≥20**
- Yarn → **pnpm ≥10** (dipin ke `pnpm@10.29.1` lewat field `packageManager`)
- Base image Docker: Debian Buster → **Debian Bookworm** (tidak ada lagi tag Ruby
  untuk Buster)
- **Redis ≥6 wajib saat boot** (bukan cuma di 7.1.3 seperti dugaan awal ROADMAP.md) —
  berlaku juga untuk task `assets:precompile`. Lihat [NOTES.md](NOTES.md) Insiden 5 & 7.
- **Skema ASCII-folding index Elasticsearch berubah**, mewajibkan
  `searchindex:rebuild` penuh meski versi ES-nya sendiri tidak naik (tetap 7.17.28,
  ≥7.8,<10). Lihat [NOTES.md](NOTES.md) Insiden 8 untuk isu index stale yang ditemukan
  saat rebuild.
- Repo paket resmi berpindah skema dari `dl.packager.io` ke `go.packager.io` — tidak
  berdampak ke kita karena build dari source, bukan lewat repo paket OS.
- **Perhatian urutan migrasi**: `CreateRecentCloses` (migrasi Nov 2025) harus
  dijalankan LEBIH DULU secara manual sebelum `db:migrate` normal, karena migrasi
  jauh lebih lama (`TaskbarAddUniquenessIndex`, Nov 2024) sudah butuh tabelnya lewat
  kode model 7.0.0. Bug urutan/desain di source Zammad sendiri — lihat
  [NOTES.md](NOTES.md) Insiden 6 untuk analisis lengkap.

## Removed

- **MySQL/MariaDB dihapus total** — gem `mysql2` sudah tidak ada di Gemfile.lock.
  PostgreSQL menjadi satu-satunya adapter didukung.
- Penghapusan integrasi Twitter & Slack.

<!-- Tambahkan di sini kalau ada perubahan skema/tabel/kolom/fitur lain yang ditemukan
selama eksekusi nyata (migrasi Rails, fitur baru/dihapus di UI, dsb). -->
