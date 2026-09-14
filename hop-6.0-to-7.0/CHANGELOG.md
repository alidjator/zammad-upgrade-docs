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

## Search / Elasticsearch

- Requirement versi tidak berubah (≥7.8, <10) — tapi **skema ASCII-folding index
  berubah**, mewajibkan `searchindex:rebuild` penuh setelah upgrade meski versi ES-nya
  sendiri tidak naik.

## Distribusi / packaging

- Repo paket resmi berpindah skema dari `dl.packager.io` ke `go.packager.io` — tidak
  berdampak ke kita karena build dari source, bukan lewat repo paket OS.

<!-- Tambahkan di sini kalau ada perubahan skema/tabel/kolom/fitur yang ditemukan
selama eksekusi nyata (migrasi Rails, fitur baru/dihapus di UI, dsb). -->
