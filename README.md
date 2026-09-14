# Zammad Upgrade Project — 3.4.0 → 7.1.3 (latest)

## Skill set yang dibutuhkan

Bukan checklist formal — ini kemampuan yang **benar-benar terpakai** selama proses hop
1-3 dan migrasi database, berdasarkan insiden nyata yang tercatat di tiap `NOTES.md`:

- **Docker & Docker Compose** — build image, debug networking antar-container (bridge
  network, `extra_hosts`/`host.docker.internal`), baca `docker compose logs`, kelola volume
- **Command line Linux** — bash dasar, `sed` untuk edit file, `systemctl`/`firewalld`,
  monitoring disk (`df`, `du`, `docker system df`) — disk penuh terjadi berkali-kali
- **Baca stack trace Ruby/Rails** — beberapa bug (mis. hop 5.0→6.0's BigDecimal issue)
  cuma bisa ditemukan akar masalahnya dengan menelusuri backtrace sampai ke kode
  ActiveRecord/gem, bukan cuma baca pesan error baris pertama
- **Administrasi database dasar** — SQL basic, beda konsep MySQL/MariaDB vs PostgreSQL,
  backup/restore (`mysqldump`, `pgloader`), grant/privilege user, **selalu verifikasi
  integritas backup** (jangan asumsikan selesai = valid)
- **Konsep dasar Elasticsearch** — cluster health, disk watermark, index lifecycle
  (drop/create/reload) — bukan expertise ES penuh, tapi cukup untuk diagnosa block/error umum
- **Git** dasar (commit, push) — dan kedisiplinan tidak commit credential asli
- **Kesabaran & pola pikir sistematis** — beberapa masalah (versi-gap MariaDB) butuh
  pivot strategi besar, bukan cuma tambal-sulam satu bug demi satu bug. Kemampuan
  mengenali kapan "tambal lagi" sudah tidak masuk akal dan perlu pendekatan berbeda
  itu lebih penting daripada hafal solusi teknis spesifik.

**Yang TIDAK wajib:** expertise mendalam di Ruby/Rails internals atau tuning ES
production-grade — sejauh ini cukup dengan riset terarah (baca source code Zammad/gem
di GitHub, cross-check dengan dokumentasi resmi) saat menemukan bug baru.

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
- Database staging: **PostgreSQL 16.13 di host** (instance yang sudah ada, dipakai bersama
  aplikasi lain dengan traffic rendah — database & user terisolasi khusus untuk Zammad).
  Sebelumnya sempat di MariaDB 10.11 container (`zammad-mariadb-legacy`, hop 4.0→6.0) karena
  MariaDB 11.8.3 host terlalu baru untuk Rails 6.0's mysql2 adapter — lihat
  [hop-4.0-to-5.0/NOTES.md](hop-4.0-to-5.0/NOTES.md). Data dimigrasikan MariaDB→PostgreSQL
  pakai `pgloader` setelah Zammad mencapai 6.0.0 — lihat [postgres-migration/NOTES.md](postgres-migration/NOTES.md).
  Database produksi asli (`zammad_production` di MariaDB 11.8.3 host) tidak disentuh sama sekali.
- Elasticsearch: container terpisah per hop (`Dockerfile.elasticsearch`), base image official Elastic.
  Versi naik seiring hop: ES 6.8.23 (hop 1) → ES 7.17.28 (hop 2, wajib karena requirement Zammad 5.0,
  masih dipakai sampai hop 3).
- Node.js: bawaan Debian per hop 1-2, **mulai hop 3 (Zammad 6.0) wajib Node.js 18.x via
  NodeSource** karena adopsi Vite (build tool JS baru) yang mensyaratkan Node ≥16.
  **Mulai hop 6.0→7.0, naik lagi ke Node.js 20.x, dan package manager JS berganti dari
  Yarn ke pnpm** (Zammad mem-pin versi pnpm lewat `package.json`, di-fetch otomatis
  pakai `corepack`).
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
| Migrasi MariaDB → PostgreSQL | ✅ **Selesai & tervalidasi** | [NOTES.md](postgres-migration/NOTES.md) (0 error, 11,3 juta baris) · [RUNBOOK.md](postgres-migration/RUNBOOK.md) |
| 6.0 → 7.0 | 🔧 Migrasi & asset selesai, reindex ES sedang berjalan | [NOTES.md](hop-6.0-to-7.0/NOTES.md) (8 insiden: pkg-config, pnpm CI=true, krisis disk, Redis ≥6, bug urutan migrasi `recent_closes`, asset pipeline 500) · [CHANGELOG.md](hop-6.0-to-7.0/CHANGELOG.md) · [RUNBOOK.md](hop-6.0-to-7.0/RUNBOOK.md) |
| 7.0 → 7.1.3 | ⬜ Belum |

## TODO — polish dokumentasi (setelah hop 7.1.3 selesai & tervalidasi)

Belum dikerjakan sekarang secara sengaja — supaya tidak mengganggu ritme dokumentasi
"catat sambil eksekusi" selama upgrade masih berjalan. Setelah seluruh proses sampai
7.1.3 selesai dan tervalidasi, terapkan:

1. Migrasi semua `CHANGELOG.md` per-hop ke format standar
   [Keep a Changelog](https://keepachangelog.com) (kategori Added/Changed/Fixed/Removed)
   — saat ini masih pakai heading bebas per topik.
2. Tambahkan daftar isi/ringkasan singkat di awal `NOTES.md` yang sudah panjang
   (terutama hop 6.0→7.0 dengan 8 insiden), supaya lebih cepat dinavigasi.
3. Review duplikasi penjelasan insiden antara `NOTES.md`/`RUNBOOK.md`/`CHANGELOG.md`
   per hop — pertimbangkan apakah perlu dipangkas atau dibiarkan (audiens beda-beda).
4. Bakukan bahasa di seluruh dokumen — saat ini masih semi-formal/informal teknis
   (mis. "kalau" → "jika/apabila", "kita" dihindari atau diganti kalimat pasif,
   "makanya"/"jadi" sebagai penghubung → "sehingga"/"oleh karena itu").

## Catatan keamanan

**Jangan simpan password asli di file manapun di folder ini.** Semua `database.yml` di sini
adalah **template** dengan placeholder — password sebenarnya hanya ada di server
(`/usr/local/src/zammad-staging/database.yml` dan `/usr/local/src/zammad-audit/database.yml`),
tidak digandakan ke laptop/Desktop ini.
