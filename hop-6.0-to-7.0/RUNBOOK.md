# Runbook — Hop 6.0 → 7.0

Langkah final yang terbukti benar di staging, hasil saringan dari [NOTES.md](NOTES.md)
(8 insiden ditemukan & diperbaiki selama eksekusi nyata). Ikuti urutan ini persis kalau
mengulang hop ini dari awal.

**Prasyarat:** hop 5.0→6.0 sudah selesai & tervalidasi, migrasi PostgreSQL sudah
selesai & tervalidasi (staging sudah berjalan di atas PostgreSQL host, bukan MariaDB).

## Pre-flight

- [ ] Cek disk space (`df -h /`, `docker system df`) — kalau di bawah ~15GB tersisa,
      bersihkan dulu SEBELUM mulai build (lihat bagian Disk di bawah). Hop ini adalah
      yang paling boros disk dari semua hop sejauh ini.
- [ ] Backup database PostgreSQL staging (`pg_dump`) sebelum mulai.

## Langkah eksekusi

**1. Clone source Zammad 7.0.0 di server**
```bash
cd /usr/local/src/zammad-staging
rm -rf app
git clone --branch 7.0.0 --depth 1 https://github.com/zammad/zammad.git app
```

**2. Sync `Dockerfile`, `Dockerfile.elasticsearch`, `docker-compose.yml` ke server**
dari folder `hop-6.0-to-7.0/` ini. **Perhatikan 3 hal wajib di Dockerfile** (kalau
menulis ulang manual, jangan sampai lupa — semua ini penyebab kegagalan nyata):
- paket `pkg-config` di daftar `apt-get install` (gem `rszr` butuh ini untuk detect
  imlib2, bukan cuma `libimlib2-dev` saja)
- `ENV CI=true` sebelum `RUN pnpm install --frozen-lockfile` (tanpa ini pnpm minta TTY
  interaktif, gagal di build non-interaktif)
- **jangan** tambahkan workaround `archive.debian.org` — Bookworm tidak membutuhkannya

Dan di `docker-compose.yml`:
- `zammad-redis` pakai `image: redis:7-alpine` (BUKAN `redis:5` — Zammad 7.0 mewajibkan
  Redis ≥6 saat boot, termasuk untuk `assets:precompile`)
- `zammad-app`, `zammad-websocket`, `zammad-scheduler` semua diberi `image:
  zammad-staging-app:7.0.0` yang SAMA (lihat langkah 4 kenapa ini kritis)
- `depends_on` ketiga service itu TIDAK lagi menyertakan `zammad-mariadb-legacy`
  (biarkan container itu tetap ada sebagai rollback safety net, cuma dilepas dari
  dependency startup)

**3. Uji `bundle install` + `pnpm install` di container sementara**
```bash
docker run --rm -v "$(pwd)/app:/opt/zammad" -w /opt/zammad ruby:3.4.8-bookworm bash -c "
  apt-get update && apt-get install -y curl gnupg git build-essential libpq-dev \
    imagemagick poppler-utils ca-certificates shared-mime-info pkg-config libimlib2-dev &&
  curl -fsSL https://deb.nodesource.com/setup_20.x | bash - &&
  apt-get install -y nodejs &&
  corepack enable &&
  CI=true pnpm install --frozen-lockfile &&
  gem install bundler -v '2.6.9' &&
  bundle install --without development test
"
```
Harus selesai dengan `Bundle complete!` (128 dependencies, 252 gems) tanpa error.

**4. Build image — HANYA SATU SERVICE, lalu `up -d` LANGSUNG (jangan ada prune di
antaranya)**

⚠️ **Kritis:** `zammad-app`/`websocket`/`scheduler` pakai Dockerfile identik. Kalau
`image:` di compose tidak dibuat sama, Compose akan build 3 image terpisah (~4-5GB x3)
DAN meng-export layer 3x secara paralel — di server dengan disk mepet ini pernah
membuat disk 100% penuh (`no space left on device`) di tengah build. Dengan `image:`
yang sama sudah diset di langkah 2, build cukup 1 service:

```bash
screen -S hop7-build
docker compose build zammad-app && docker compose up -d
```

⚠️ **Jangan** jalankan `docker image prune -af` di antara `build` dan `up -d` — pernah
menghapus image yang baru saja dibuild (dianggap "belum terpakai" karena belum ada
container yang memakainya), memaksa build ulang dari nol (~12 menit terbuang).

**5. Verifikasi container stabil (tidak crash-loop)**
```bash
docker compose ps
docker compose logs zammad-app --tail 50
```
Kalau ada `Error: incompatible Redis version` di log dan container terus "Restarting"
— pastikan `zammad-redis` sudah pakai image `redis:7-alpine`, bukan `redis:5`.

**6. Migrasi database — WAJIB urut khusus karena bug migrasi resmi Zammad**
```bash
# Migrasi ini harus dijalankan LEBIH DULU, di luar urutan normal — lihat NOTES.md
# Insiden 6 untuk penjelasan lengkap kenapa ini aman dan perlu.
docker compose exec zammad-app env RAILS_ENV=production bundle exec rake db:migrate:up VERSION=20251106095318

docker compose exec zammad-app env RAILS_ENV=production bundle exec rake db:migrate
```
Verifikasi bersih:
```bash
docker compose exec zammad-app env RAILS_ENV=production bundle exec rake db:migrate:status | grep -v "^   up"
```
(harus kosong, tidak ada baris `down`)

**7. Verifikasi asset pipeline benar-benar ter-precompile — JANGAN cuma percaya status
container "Up"**

Container bisa "Up" dan `rails server` jalan normal, TAPI kalau `assets:precompile`
gagal diam-diam di boot pertama (misal karena race dengan crash-loop Redis di langkah
sebelumnya), semua halaman akan 500. Selalu verifikasi:
```bash
docker compose exec zammad-app ls public/assets/ | grep -E "application-.*\.css"
curl -sI http://localhost:3010/
```
Kalau `curl` mengembalikan 500 atau tidak ada file `application-*.css`, jalankan
precompile manual lalu restart:
```bash
docker compose exec zammad-app env RAILS_ENV=production bundle exec rake assets:precompile
docker compose restart zammad-app
curl -sI http://localhost:3010/   # harus 200 OK
```

**8. Rebuild search index — cek dulu index stale sebelum mulai**
```bash
docker compose exec zammad-elasticsearch curl -s "http://localhost:9200/_cat/indices?v"
```
Kalau ada index `localhost.localdomain_zammad_production_*` yang seharusnya sudah tidak
ada (dari percobaan rebuild sebelumnya yang gagal di tengah jalan), hapus dulu:
```bash
docker compose exec zammad-elasticsearch curl -s -X DELETE "http://localhost:9200/localhost.localdomain_zammad_production_*"
```
Baru jalankan rebuild (lama, jalankan di `screen`):
```bash
screen -S hop7-reindex
docker compose exec zammad-app env RAILS_ENV=production bundle exec rake zammad:searchindex:rebuild
```
Kalau gagal lagi dengan `resource_already_exists_exception`, ulangi cleanup index di
atas dan coba lagi dari kondisi benar-benar bersih.

**9. Verifikasi akhir**
- [ ] UI menampilkan versi 7.0.x, login & buka tiket normal
- [ ] Search tiket berfungsi (validasi hasil rebuild index)
- [ ] WebSocket real-time update jalan (cek `/cable` di browser devtools)
- [ ] `db:migrate:status` semua `up`

## Disk — pembersihan aman sebelum/selama hop ini

Hop ini paling boros disk dari semua hop (base image lebih besar + Vite build + build
tripel kalau langkah 4 di atas tidak diikuti). Urutan pembersihan aman yang terbukti:

```bash
docker builder prune -af
docker rmi <image-lama-yang-sudah-dikonfirmasi-tidak-dipakai>   # cek dulu docker ps -a
docker volume rm <volume-anonymous-kosong>                       # cek dulu docker volume inspect
```
**Jangan** pakai `docker image prune -af` (hapus SEMUA image tak terpakai tanpa
pandang bulu) di server bersama — selalu hapus by-name setelah verifikasi manual.

## Kalau gagal / perlu mundur

`zammad-mariadb-legacy` tidak lagi dipakai service manapun di hop ini, tapi datanya
masih utuh sebagai jaring pengaman terakhir. Untuk mundur satu langkah (ke image
6.0.0), checkout ulang folder `app/` ke tag `6.0.0` dan rebuild — data PostgreSQL
tidak berubah struktur secara merusak oleh migrasi yang gagal di tengah jalan (migrasi
Rails idempoten per-file, cek `db:migrate:status` untuk tahu titik terakhir yang
berhasil).
