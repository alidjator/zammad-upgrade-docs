# Glosarium

Istilah yang dipakai berulang di seluruh dokumentasi proyek ini, dengan asumsi
pembaca familiar dengan Docker/Rails/Elasticsearch dasar tapi belum tentu dengan
istilah spesifik proyek ini atau operasional Zammad.

**Hop** — satu lompatan upgrade major version Zammad (mis. `3.4.0 → 4.0`). Istilah
proyek ini, bukan istilah resmi Zammad. Lihat [ROADMAP.md](ROADMAP.md) untuk daftar
lengkap hop dan requirement per hop.

**Insiden** — masalah nyata yang ditemukan & diperbaiki selama eksekusi sebuah hop,
didokumentasikan secara kronologis di `NOTES.md` masing-masing hop. Lihat
[INCIDENT_INDEX.md](INCIDENT_INDEX.md) untuk indeks semua insiden lintas-hop.

**Sandbox riset** — seluruh environment di server `Koi-Server-Dev` (termasuk stack
`zammad-audit` dan `zammad-staging`), terisolasi penuh dari produksi nyata. Lihat
[README.md § Konteks](README.md#konteks).

**"Produksi" (bertanda kutip)** — istilah proyek ini untuk stack `zammad-audit` yang
berperan mensimulasikan produksi *di dalam* sandbox riset — BUKAN sistem produksi
sungguhan. Tanda kutip sengaja dipakai konsisten di seluruh dokumen untuk membedakan
dari produksi nyata (yang berjalan di infrastruktur terpisah).

**Reindex** — proses membangun ulang index pencarian Elasticsearch dari data
database (rake task `searchindex:rebuild` / `zammad:searchindex:rebuild`,
namanya berubah-ubah antar versi Zammad — lihat [INCIDENT_INDEX.md](INCIDENT_INDEX.md)).
Biasanya tahap paling lama dari seluruh proses upgrade — lihat
[DOWNTIME_ESTIMATE.md](DOWNTIME_ESTIMATE.md).

**Flood-stage watermark** — ambang batas disk Elasticsearch (default 95% penggunaan)
yang membuat ES otomatis mengunci index jadi read-only (`read_only_allow_delete`).
Block ini **TIDAK otomatis lepas** hanya karena disk dibersihkan — harus dibuka manual
lewat `_cluster/settings` API, dan baru benar-benar tidak muncul lagi jika disk di
bawah *high watermark* (90%, bukan 95%).

**High/low watermark** — ambang disk Elasticsearch yang lebih longgar (default
90%/85%) yang mengatur alokasi shard baru — beda dari flood-stage watermark, tidak
mengunci index jadi read-only.

**`screen` / `tmux`** — terminal multiplexer, dipakai untuk proses panjang (build
image, reindex ES) supaya tidak terputus jika koneksi SSH bermasalah. Lihat
[DOWNTIME_ESTIMATE.md § "Standardisasi metode pengukuran"](DOWNTIME_ESTIMATE.md#standardisasi-metode-pengukuran-mulai-hop-70713)
untuk kenapa proyek ini tidak mengandalkan `time` di dalam `screen`.

**pgloader** — tool open-source yang dipakai untuk migrasi data MariaDB →
PostgreSQL. Lihat [postgres-migration/](postgres-migration/) untuk detail lengkap.

**`assets:precompile`** — rake task Rails yang mengompilasi CSS/JS jadi asset
statis. Di proyek ini WAJIB dijalankan saat runtime (`command:` di
docker-compose.yml), bukan saat build time, karena butuh koneksi database yang baru
tersedia setelah container jalan.

**ASCII-folding** — fitur normalisasi teks pencarian Elasticsearch (mis. pencarian
"café" ikut mencocokkan "cafe"). Perubahan pada fitur ini di salah satu rilis Zammad
mewajibkan rebuild index penuh, bukan cuma restart.

**`CI=true`** — environment variable yang harus di-set sebelum `pnpm install` di
dalam Dockerfile, supaya pnpm tidak meminta TTY interaktif dan build non-interaktif
tidak gagal dengan `ERR_PNPM_ABORTED_REMOVE_MODULES_DIR_NO_TTY`.

**packager.io** — layanan hosting paket resmi Zammad. Skema domainnya berubah dari
`dl.packager.io` ke `go.packager.io` di salah satu rilis — tidak relevan untuk
proyek ini karena build dari source (`git clone`), bukan dari paket resmi.

**`RAILS_ENV=production`** — mode environment Rails bawaan. **Ini bukan indikasi
lingkungan produksi nyata** — index Elasticsearch sandbox riset ini tetap berprefix
`zammad_production` semata-mata karena mencerminkan environment variable ini,
sesuai konvensi Rails/Zammad standar. Lihat [README.md § Konteks](README.md#konteks).

**Keep a Changelog** — format standar (Added/Changed/Deprecated/Removed/Fixed/
Security) yang dipakai semua `CHANGELOG.md` per-hop di proyek ini. Lihat
[keepachangelog.com](https://keepachangelog.com).

**Diátaxis** — framework dokumentasi (Tutorial/How-to guide/Reference/Explanation)
yang jadi acuan struktur dokumentasi proyek ini. Lihat
[PRODUCTION_READINESS_TODO.md § 9](PRODUCTION_READINESS_TODO.md#9-kelengkapan-gaya-di)
dan [diataxis.fr](https://diataxis.fr).
