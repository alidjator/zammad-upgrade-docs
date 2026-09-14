# Hop 5.0 → 6.0 — Catatan Lengkap (Status: ✅ Selesai & Tervalidasi)

## Ringkasan

Hop ini paling banyak perubahan arsitektur runtime dari semua hop sejauh ini: rename
script scheduler, Redis jadi hard dependency dengan config eksplisit, dan adopsi Vite
(build tool JS modern) yang butuh Node.js jauh lebih baru + yarn. Tidak ada masalah gem
Ruby sama sekali (`bundle install` mulus dari awal, beda dari hop 1).

## Perubahan versi

- Ruby: 2.7.4 → **3.1.3** (base image tetap Debian Buster, tag tersedia)
- Bundler: 2.2.20 → **2.4.1**
- Rails: 6.0.4.1 → **6.1.7.3**
- Node.js: **10.24 (bawaan Buster) → 18.x LTS via NodeSource** (wajib — package.json
  Zammad 6.0.0 mensyaratkan `"node": ">=16"`)
- Elasticsearch, Database: tidak berubah (tetap ES 7.17.28, MariaDB 10.11 legacy)

## Masalah yang ditemukan & fix-nya (urutan kejadian)

### 1. Lupa transfer file Dockerfile/docker-compose.yml baru ke server
Kesalahan proses, bukan bug Zammad — file baru cuma tersimpan di laptop, server masih
pakai Dockerfile hop sebelumnya (`ruby:2.7.4-buster`). **Pelajaran:** selalu berikan
command `cat > ... << 'EOF'` eksplisit untuk menulis file di server, jangan asumsikan
sudah tersalin.

### 2. `script/scheduler.rb` di-rename jadi `script/background-worker.rb`
Command CLI juga berubah dari `run` jadi `start`. Arsitektur background job jadi lebih
terpusat (`BackgroundServices::Cli`, mengelola beberapa service sekaligus, bukan cuma
scheduler tunggal).

**Fix di docker-compose.yml:**
```yaml
command: ["bash", "-c", "bundle exec ruby script/background-worker.rb start"]
```
Juga perlu env var `BACKGROUND_SERVICES_LOG_TO_STDOUT: '1'` — defaultnya sekarang TIDAK
log ke stdout (beda dari versi lama), jadi `docker compose logs` akan kosong tanpa ini.

### 3. Redis jadi hard dependency — gagal connect ke `localhost:6379`
```
Redis::CannotConnectError: Error connecting to Redis on localhost:6379
```
Sebelumnya Redis dipakai tapi tidak divalidasi ketat saat boot. Mulai 6.0, aplikasi
langsung mencoba connect ke Redis saat startup, dan defaultnya mengarah ke `localhost`
(tidak relevan di Docker Compose, karena Redis ada di container terpisah).

**Fix:** tambahkan `REDIS_URL: redis://zammad-redis:6379` ke environment tiap service
(app, websocket, scheduler).

### 4. Vite build gagal — `Errno::ENOENT: yarn`
Zammad 6.0 mengadopsi Vite sebagai build tool JS tambahan (selain Sprockets yang sudah
ada). `assets:precompile` sekarang juga menjalankan task `vite:build_all`, yang butuh:
- **Node.js ≥16** (Buster bawaan cuma 10.x) — install via NodeSource:
  ```bash
  curl -fsSL https://deb.nodesource.com/setup_18.x | bash -
  apt-get install -y nodejs
  ```
- **yarn** (untuk `vite_ruby` mengeksekusi build) — `npm install -g yarn`
- **`node_modules` ter-populate** — `yarn install --frozen-lockfile` di dalam `app/`,
  dijalankan di **build time** (bukan runtime seperti `assets:precompile` — `yarn
  install` tidak butuh koneksi database, jadi aman di Dockerfile langsung, mempercepat
  startup container)

### 5. Nama rake task search index berubah LAGI (3 kali berubah, 3 hop berturut-turut)
Sekarang balik ke `zammad:searchindex:` (dengan prefix `zammad:`), dan sekarang PUNYA
deskripsi jadi muncul normal di `rake --tasks` (di hop 1 & 2 harus cek source langsung
karena tidak muncul). **Pelajaran permanen:** jangan pernah asumsikan nama/namespace task
ini sama antar versi — selalu cek ulang tiap hop.

### 6. Disk penuh lagi (95% → butuh cleanup 6,5GB build cache)
Pola yang sama seperti 2 hop sebelumnya. Kali ini build image jauh lebih besar dari
biasanya karena `node_modules` (proses build image ada baris "transferring context:
1.11GB" — jauh lebih besar dari hop-hop sebelumnya yang cuma puluhan-ratusan MB).
**Fix:** `docker builder prune -af` (membebaskan 6,5GB).

### 7. Index ES nyangkut lagi dari percobaan gagal (partial reload)
Setelah retry pertama gagal karena disk, index `ticket` menunjukkan **743.560 dokumen**
(jauh dari 161.884 tiket asli) sementara `user` masih 0 — indikasi reload berhenti di
tengah proses `Ticket` (kemungkinan termasuk nested document `ticket_article` yang
dihitung terpisah oleh ES). **Fix:** hapus semua index Zammad manual sebelum retry penuh
(pola sama seperti hop sebelumnya).

## Durasi (jauh lebih lama dari hop-hop sebelumnya)

- **Migrasi schema:** 9m25s (150+ migrasi — assumsi migrasi setahun 2021-2023 semua masuk
  di sini karena kita loncat 5.0.0 langsung ke 6.0.0). Dua migrasi berat:
  `RemoveDuplicateTranslations` (160,7s) dan `TaskbarUpdatePreferenceTasks` (387,6s/~6,5 menit)
- **Reindex ES:** 337m47s (~5,6 jam) — `Ticket` sendiri 18.260s (~5,1 jam), `User` 1.790s
  (~29,8 menit). Ini yang paling lama dari semua hop sejauh ini.

## Verifikasi akhir

- ✅ Admin Panel → System → Version menunjukkan "Zammad version 6.0.0"
- ✅ Search berfungsi (50 tiket, 15 user, 2 KB answer untuk kata kunci uji)
- ✅ Console browser cuma warning ringan yang sudah dikenal (sync XHR deprecation)
