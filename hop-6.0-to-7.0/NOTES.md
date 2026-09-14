# Hop 6.0 → 7.0 — Catatan Lengkap (Status: 🔧 Migrasi & asset selesai, reindex ES sedang berjalan)

## Ringkasan requirement (hasil riset awal, tervalidasi lewat eksekusi nyata)

| Item | Versi | Perubahan dari hop 6.0 |
|---|---|---|
| Ruby | 3.4.8 | naik dari 3.1.3 |
| Bundler | 2.6.9 | naik dari 2.4.1 |
| Rails | 8.0.4 | **loncat dari 6.1.7.3 — Rails 7 dilewati total** |
| Node.js | ≥20 | naik dari ≥16 |
| Package manager JS | **pnpm ≥10** (`packageManager: "pnpm@10.29.1"`) | **ganti dari Yarn** |
| Database | PostgreSQL saja | mysql2 sudah hilang total dari Gemfile.lock |
| Base OS image | Debian Bookworm (`ruby:3.4.8-bookworm`) | naik dari Buster — tidak ada tag `3.4.8-buster` |
| Redis | **≥6 wajib saat boot** (ditemukan lewat crash loop nyata, bukan dari riset awal) | naik dari `redis:5` |
| Elasticsearch | ≥7.8, <10 (tidak berubah versi) | tetap 7.17.28, TAPI wajib rebuild index (ASCII-folding) |

## Insiden 1 — `bundle install` gagal: gem `rszr` butuh `pkg-config`

Ditemukan saat uji coba `bundle install` di container sementara (`ruby:3.4.8-bookworm`),
sebelum menulis Dockerfile final: `checking for pkg-config for imlib2... not found`.
Root cause: paket `pkg-config` sendiri tidak ter-install oleh Debian secara default —
`libimlib2-dev` sudah cukup untuk header, tapi tanpa binary `pkg-config`, extconf gagal
mendeteksinya. **Fix:** tambahkan `pkg-config` ke daftar `apt-get install` di
[Dockerfile](Dockerfile). Setelah fix: `bundle install` sukses, 128 dependencies/252 gems.

## Insiden 2 — Debian Bookworm TIDAK butuh workaround archive.debian.org

Beda dari semua image Buster (hop 1-3) yang selalu butuh redirect ke
`archive.debian.org` karena EOL, `apt-get update` di Bookworm berjalan mulus tanpa
modifikasi apa pun — Bookworm (rilis 2023) masih dalam masa dukungan resmi Debian saat
hop ini dieksekusi (2026).

## Insiden 3 — `pnpm install` gagal di Docker build: butuh `CI=true`

`docker compose build` gagal dengan `ERR_PNPM_ABORTED_REMOVE_MODULES_DIR_NO_TTY` —
pnpm minta konfirmasi TTY interaktif untuk menghapus `node_modules`, mustahil dipenuhi
di build non-interaktif. **Fix:** tambahkan `ENV CI=true` sebelum `RUN pnpm install
--frozen-lockfile` di Dockerfile (pesan error pnpm sendiri menyarankan solusi ini).
Env var ini persisten ke container runtime juga, jadi turut membantu saat
`assets:precompile` runtime memanggil pnpm/Vite lagi (lihat Insiden 6).

## Insiden 4 — Krisis disk berulang selama build (paling parah di hop ini)

Staging disk (`/`, 130GB) berulang kali mepet 93-100% selama proses build:

- **Root cause utama:** `zammad-app`, `zammad-websocket`, `zammad-scheduler` punya
  Dockerfile identik tapi tidak diberi `image:` yang sama di `docker-compose.yml` —
  Compose/buildx membangun **3 image terpisah** (~4-5GB masing-masing) dan bahkan meng-
  **export layer 3 kali secara paralel** meski akhirnya ditag sama, sempat membuat disk
  benar-benar 100% penuh (`no space left on device` di tengah proses).
- **Fix permanen:** beri `image: zammad-staging-app:7.0.0` yang SAMA ke ketiga service
  di [docker-compose.yml](docker-compose.yml), lalu build **hanya satu service**
  (`docker compose build zammad-app`) sebelum `docker compose up -d` — Compose otomatis
  memakai image yang sama untuk service lain tanpa build ulang. Ini pola yang seharusnya
  diterapkan juga ke hop-hop sebelumnya kalau upgrade diulang dari awal.
- **Insiden tambahan yang HARUS diwaspadai:** `docker image prune -af` yang dijalankan
  SEBELUM `docker compose up -d` (saat image baru belum dipakai container manapun)
  **menghapus image yang baru saja di-build**, karena dari sudut pandang Docker image
  itu "belum terpakai". **Pelajaran:** setelah `docker compose build`, LANGSUNG lanjut
  `docker compose up -d` tanpa command prune apa pun di antaranya. `prune -af` command
  itu juga sempat menghapus beberapa image milik tooling lain di server bersama ini
  (gitlab-runner-helper, aquasec/trivy, python:3.11-alpine, docker:29.0.4/dind) — resiko
  rendah (bukan container yang sedang jalan, cuma perlu re-pull saat dipakai lagi) tapi
  tetap jadi pengingat: **jangan jalankan `prune -a` di server bersama tanpa review
  daftar image dulu**, cukup hapus by-name yang sudah dikonfirmasi tidak dipakai.
- Pembersihan aman yang dilakukan berulang kali: `docker builder prune -af`,
  `docker rmi` spesifik untuk image base lama tidak terpakai (`ruby:3.1.3-buster`,
  `ruby:2.7.4-buster`, `mariadb:11`, `dimitri/pgloader`), `docker volume rm` untuk
  anonymous volume kosong (93B, sisa container `--rm` yang mungkin tidak bersih
  sempurna).

## Insiden 5 — Redis 5 tidak lagi didukung (ditemukan lewat crash loop)

Setelah image berhasil dibuild dan container di-`up`, `zammad-app`/`websocket`/
`scheduler` semua crash-loop dengan `Error: incompatible Redis version (6+ required;
5.0.14 found)`. ROADMAP.md sebelumnya mencatat "Redis ≥6 wajib" hanya di baris 7.1.3,
ternyata requirement ini **sudah berlaku sejak 7.0.0**. **Fix:** naikkan
`zammad-redis` dari `redis:5` ke `redis:7-alpine` di docker-compose.yml. Redis di sini
cuma dipakai untuk cache/pub-sub ActionCable (tidak ada volume/data persisten), jadi
aman diganti langsung tanpa migrasi data.

## Insiden 6 — Bug urutan migrasi resmi Zammad: `recent_closes` belum ada saat dibutuhkan

`rake db:migrate` berhenti di migrasi `20241106073757 TaskbarAddUniquenessIndex` dengan
error `PG::UndefinedTable: relation "recent_closes" does not exist`. Root cause:
migrasi ini men-dedup baris `Taskbar` duplikat sebelum menambah unique index, memanggil
`.destroy` pada baris duplikat — ini memicu callback `after_destroy_commit
:log_recent_close` di model `Taskbar` (kode model TERBARU dari tag 7.0.0, sudah
memuat fitur `RecentClose` yang baru ditambahkan di masa depan relatif terhadap tanggal
migrasi ini). Tabel pendukungnya baru dibuat migrasi `20251106095318 CreateRecentCloses`
— **lebih dari setahun kemudian** dalam urutan migrasi. Ini murni bug urutan/desain di
source Zammad sendiri (muncul karena kita menjalankan puluhan migrasi historis
sekaligus dengan model code final, bukan inkremental per-rilis seperti alur upgrade
normal), BUKAN kesalahan konfigurasi kita.

**Fix aman (tanpa modifikasi source Zammad):** verifikasi dulu bahwa migrasi
`CreateRecentCloses` benar-benar berdiri sendiri (cuma `create_table` + registrasi
scheduler job, tidak bergantung migrasi lain di antaranya) — konfirmasi dengan
membaca isi filenya langsung. Setelah yakin aman, jalankan migrasi itu duluan di luar
urutan:
```bash
docker compose exec zammad-app env RAILS_ENV=production bundle exec rake db:migrate:up VERSION=20251106095318
docker compose exec zammad-app env RAILS_ENV=production bundle exec rake db:migrate
```
Setelah itu seluruh 78 migrasi tersisa (rentang November 2024 - Februari 2026) berjalan
lancar tanpa error lain.

## Insiden 7 — Error 500 pasca-migrasi: asset pipeline tidak pernah ter-precompile

Setelah migrasi selesai dan container `zammad-app` akhirnya bisa boot (pasca fix Redis),
`helpdesk.satu.solutions` (dan `curl` langsung ke container) menampilkan **500 Internal
Server Error** — anehnya halaman error itu sendiri menampilkan tag ERB mentah yang
tidak ter-render (`<% if @traceback %>`), bukan HTML biasa.

**Root cause (dikonfirmasi dari `log/production.log`):**
1. CMD container adalah `rm -f tmp/pids/server.pid; assets:precompile; rails server`
   — tiga perintah disambung `;` (bukan `&&`), jadi tetap lanjut walau perintah
   sebelumnya gagal.
2. `assets:precompile` JUGA mem-boot seluruh environment Rails (bukan cuma compile
   file) — jadi ikut kena blokir cek Redis ≥6 yang sama seperti `rails server`
   (Insiden 5). Selama Redis masih `redis:5`, precompile gagal instan tanpa menulis
   satu pun file, lalu `rails server` juga gagal dengan alasan sama → container
   restart terus (crash loop).
3. Begitu Redis diperbaiki, `rails server` akhirnya berhasil boot di salah satu siklus
   restart — tapi `assets:precompile` pada siklus yang sama TIDAK sempat menghasilkan
   file compiled (tidak ada log "Writing ..." untuk boot yang berhasil itu). Hasilnya:
   server jalan, tapi `public/assets/` kosong (tidak ada `manifest.json` atau
   `application-*.css`).
4. Setiap request ke `/` gagal dengan `Sprockets::Rails::Helper::AssetNotFound: The
   asset "application.css" is not present in the asset pipeline` di
   `app/views/layouts/application.html.erb:9` (`stylesheet_link_tag "application"`).
5. **Bug kedua yang memperparah:** saat Zammad mencoba menampilkan halaman error yang
   informatif untuk exception di atas, `ApplicationController::HandlesErrors
   #respond_to_exception` sendiri melempar exception KEDUA
   (`ActionController::RespondToMismatchError` — "respond_to was called multiple times
   and matched with conflicting formats"). Ini membuat Rails jatuh ke fallback paling
   akhir: membaca `public/500.html` **mentah sebagai file statis tanpa diproses ERB**
   (Rack tidak pernah meng-eval ERB untuk static fallback) — makanya tag `<% %>` muncul
   apa adanya di response, bukan pesan error yang manusiawi.

**Fix:** jalankan `assets:precompile` manual saat Redis sudah sehat (`docker compose
exec zammad-app env RAILS_ENV=production bundle exec rake assets:precompile`) — kali
ini sukses penuh (~5 menit: compile CSS Sprockets + build Vite lengkap dengan
`pnpm`/`corepack`, ~230 chunk JS + manifest.json). Karena Rails meng-cache manifest
asset di memori sekali saat boot, **restart container wajib** (`docker compose restart
zammad-app`) supaya Puma memuat ulang manifest yang baru ada. Restart kedua ini cepat
("Skipping vite build. Watched files have not changed") karena Vite mendeteksi tidak
ada perubahan sejak build manual barusan. Setelah restart: `curl` mengonfirmasi
`200 OK`.

**Pelajaran untuk hop berikutnya (7.0→7.1.3) dan dokumentasi produksi nanti:** kalau
`assets:precompile` gagal sekali karena alasan APAPUN (bukan cuma Redis) saat boot
otomatis, container akan tetap "Up" (karena `rails server` bisa jalan independen) tapi
500 di semua halaman — **selalu verifikasi `public/assets/` benar-benar berisi
`application-*.css` dan `manifest.json` setelah container pertama kali `Up`**, jangan
asumsikan sukses hanya dari status container.

## Insiden 8 — Index Elasticsearch stale menghalangi `searchindex:rebuild`

Task `zammad:searchindex:rebuild` gagal di tahap `Creating indexes` dengan
`resource_already_exists_exception` untuk index `..._group`, padahal tahap sebelumnya
("Dropping indexes... done") melaporkan sukses. Cek `_cat/indices` mengonfirmasi 6
index `zammad_production_*` masih ada (kemungkinan race condition — background job
sempat menulis/membuat index di antara langkah drop dan create, atau drop tidak
menyasar index yang dibuat lazy oleh model baru seperti `ai_agent`/`core_workflow`).
Pola ini sudah pernah terjadi di hop-hop sebelumnya (dicatat di ROADMAP.md sebagai
"masalah yang berulang"). **Fix:** hapus manual semua index `zammad_production_*` via
API ES (`DELETE /localhost.localdomain_zammad_production_*`), verifikasi cuma
`.geoip_databases` (index sistem, bukan punya Zammad) yang tersisa, baru jalankan
ulang `searchindex:rebuild` dari kondisi benar-benar bersih.

## Verifikasi migrasi database

- ✅ `db:migrate` — 78 migrasi berjalan lancar (setelah fix urutan `recent_closes`),
  `db:migrate:status` bersih (semua "up")
- Dataset pasca-migrasi konsisten dengan hasil migrasi Postgres sebelumnya: 161.894
  tiket, 1.031.898 artikel, 72.014 user, 1.851 organisasi (bertambah sedikit dari
  aktivitas staging sejak migrasi Postgres — bukan kehilangan data)

## Status validasi hop ini

- ✅ `pnpm install` + `bundle install` — 0 error (setelah fix pkg-config, 128
  dependencies/252 gems)
- ✅ Build image (strategi 1 image dipakai 3 service) — sukses, ~12 menit
- ✅ `db:migrate` — 78 migrasi sukses (setelah fix urutan recent_closes)
- ✅ `assets:precompile` — sukses manual, halaman web `200 OK` setelah restart
- 🔧 `searchindex:rebuild` — sedang berjalan ulang setelah cleanup index stale
- ⬜ Verifikasi UI/search penuh — menunggu reindex selesai
