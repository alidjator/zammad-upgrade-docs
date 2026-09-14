# Changelog — Hop 7.0 → 7.1.3

Sumber: `Gemfile.lock`, `package.json`, `BREAKING_CHANGES.md` langsung di tag `7.1.3`
GitHub `zammad/zammad`.

## Environment

- Ruby 3.4.8 → **3.4.9** (patch)
- Rails 8.0.4 → **8.0.5.1** (patch)
- Node.js ≥20 → **≥24**
- pnpm dipin `10.29.1` → **`10.33.3`** (otomatis lewat corepack)
- Bundler, base OS (Debian Bookworm), database (PostgreSQL) — tidak berubah

## Breaking changes resmi (dari BREAKING_CHANGES.md § 7.1)

- **Elasticsearch 7 dinyatakan deprecated** — belum jadi hard requirement di 7.1.3,
  tapi versi Zammad setelahnya akan mewajibkan ES 8+.
- **Calendar iCal feed wajib URL HTTP/HTTPS** — path file lokal tidak lagi didukung.
  Tidak relevan untuk instance ini (tidak memakai fitur calendar iCal file lokal).
- **`Exceptions::UnprocessableEntity` deprecated**, diganti `Exceptions::UnprocessableContent`
  (akan dihapus di Zammad 8.0). Tidak relevan — tidak ada integrasi custom yang
  memanggil exception class ini secara langsung.

<!-- Tambahkan di sini kalau ada perubahan skema/tabel/kolom/fitur yang ditemukan
selama eksekusi nyata. -->
