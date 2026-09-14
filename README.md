# Zammad Upgrade Project — 3.4.0 → 7.1.3 (latest)

## Konteks

Self-hosted Zammad di server `Koi-Server-Dev` (CentOS Stream 9), berjalan via Docker Compose
custom (bukan install native, bukan image resmi Zammad). Domain produksi: `helpdesk.satu.solutions`
(reverse proxy nginx, SSL Let's Encrypt sudah ada).

**Selama proses upgrade ini berlangsung (sampai hop 7 selesai), domain produksi sengaja
diarahkan ke environment staging.** Produksi asli (project Docker Compose `zammad-audit` di
`/usr/local/src/zammad-audit`) dimatikan sementara untuk membebaskan resource server.

## Arsitektur

- App: custom Dockerfile (base image Ruby berubah per hop sesuai requirement) + source Zammad
  dari git clone per versi
- Database staging: **container terpisah `zammad-mariadb-legacy` (MariaDB 10.11)** — sejak
  hop 4.0→5.0, BUKAN lagi MariaDB 11.8.3 di host. Alasan: MariaDB 11.8.3 terlalu baru,
  menyebabkan banyak bug inkompatibilitas dengan Rails 6.0's mysql2 adapter. Lihat
  [hop-4.0-to-5.0/NOTES.md](hop-4.0-to-5.0/NOTES.md) bagian "Keputusan besar".
  Database produksi asli (`zammad_production` di MariaDB 11.8.3 host) tidak disentuh.
- Elasticsearch: container terpisah per hop (`Dockerfile.elasticsearch`), base image official Elastic.
  Versi naik seiring hop: ES 6.8.23 (hop 1) → ES 7.17.28 (hop 2, wajib karena requirement Zammad 5.0,
  masih dipakai sampai hop 3).
- Node.js: bawaan Debian per hop 1-2, **mulai hop 3 (Zammad 6.0) wajib Node.js 18.x via
  NodeSource** karena adopsi Vite (build tool JS baru) yang mensyaratkan Node ≥16.
- Reverse proxy: nginx di host (bukan container), config di `/etc/nginx/conf.d/helpdesk.satu.solutions.conf`.

## Lokasi kerja di server

- Produksi (mati sementara): `/usr/local/src/zammad-audit`
- Staging (working copy, project name `zammad-staging`): `/usr/local/src/zammad-staging`

## Roadmap upgrade

Zammad **tidak boleh loncat major version** — urutan wajib:

```
3.4.0 → 4.0 → 5.0 → 6.0 → [migrasi MariaDB→PostgreSQL, wajib ≥5.3] → 7.0 → 7.1.3 (latest)
```

Detail requirement per hop ada di [ROADMAP.md](ROADMAP.md).

## Status

| Hop | Status | Catatan |
|---|---|---|
| 3.4.0 → 4.0 | ✅ **Selesai & tervalidasi** | [NOTES.md](hop-3.4.0-to-4.0/NOTES.md) · [CHANGELOG.md](hop-3.4.0-to-4.0/CHANGELOG.md) · [RUNBOOK.md](hop-3.4.0-to-4.0/RUNBOOK.md) |
| 4.0 → 5.0 | ✅ **Selesai & tervalidasi** | [NOTES.md](hop-4.0-to-5.0/NOTES.md) (keputusan pindah ke MariaDB 10.11) · [CHANGELOG.md](hop-4.0-to-5.0/CHANGELOG.md) · [RUNBOOK.md](hop-4.0-to-5.0/RUNBOOK.md) |
| 5.0 → 6.0 | ✅ **Selesai & tervalidasi** | [NOTES.md](hop-5.0-to-6.0/NOTES.md) (Redis hard dependency, Vite/Node.js 18) · [CHANGELOG.md](hop-5.0-to-6.0/CHANGELOG.md) · [RUNBOOK.md](hop-5.0-to-6.0/RUNBOOK.md) |
| Migrasi MariaDB → PostgreSQL | 🔜 Belum dimulai — **sudah bisa dimulai sekarang** (Zammad 6.0.0 ≥5.3, syarat tool resmi terpenuhi) |
| 6.0 → 7.0 | ⬜ Belum (setelah migrasi Postgres) |
| 7.0 → 7.1.3 | ⬜ Belum |

## Catatan keamanan

**Jangan simpan password asli di file manapun di folder ini.** Semua `database.yml` di sini
adalah **template** dengan placeholder — password sebenarnya hanya ada di server
(`/usr/local/src/zammad-staging/database.yml` dan `/usr/local/src/zammad-audit/database.yml`),
tidak digandakan ke laptop/Desktop ini.
