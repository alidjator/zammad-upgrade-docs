# Hop 5.0 → 6.0 — Belum Dimulai

## Requirement (lihat [../ROADMAP.md](../ROADMAP.md) untuk detail lengkap)

- Ruby: 2.7.4 → **3.1.3**
- Rails: kemungkinan naik dari 6.0.x ke 6.1.x (cek `Gemfile.lock` setelah clone 6.0.0)
- Elasticsearch: masih ≥7.8, <9 — ES 7.17.28 yang sudah dipakai kemungkinan masih cukup,
  tapi tetap cek requirement resmi 6.0
- Database: **tetap pakai MariaDB 10.11 di container `zammad-mariadb-legacy`** — JANGAN
  kembali ke MariaDB 11.8.3 host. Bug versi-gap yang ditemukan di hop 4.0→5.0
  (lihat [../hop-4.0-to-5.0/NOTES.md](../hop-4.0-to-5.0/NOTES.md)) kemungkinan besar masih
  relevan karena masih Rails 6.x.
- **Redis jadi hard dependency mulai versi ini** (sudah ada di compose, tapi cek requirement
  versi Redis minimalnya)
- **Reverse-proxy nginx perlu disesuaikan untuk WebSocket/ActionCable** (`/cable`) — ini
  BARU muncul di hop ini, belum pernah dilakukan sebelumnya. Cek config nginx
  (`/etc/nginx/conf.d/helpdesk.satu.solutions.conf`) apakah location `/ws` yang sudah ada
  masih cukup atau perlu ditambah location `/cable` terpisah.

## Pola masalah yang sudah terbukti berulang (cek dari awal, sebelum build penuh)

1. **Cek dulu di container sementara** (`docker run --rm -v .../app:/opt/zammad ...`)
   apakah `bundle install` mulus sebelum masuk ke Dockerfile permanen.
2. **`assets:precompile` di CMD/command runtime, bukan di Dockerfile build** — pola ini
   sudah permanen sejak hop sebelumnya, pastikan Dockerfile baru tetap ikuti pola ini.
3. **Cek nama rake task search index lagi** — sudah 2x hop, namespace `searchindex:` (bukan
   `zammad:searchindex:`), tapi tidak menjamin tidak berubah lagi.
4. **Kalau ada bug versi-gap database muncul lagi** — coba dulu fresh clone tanpa patch
   sebelum menambal manual, untuk pastikan bukan sisa masalah lama.
5. **Pantau disk terus** — `docker builder prune -af` setelah beberapa kali build gagal,
   jangan tunggu sampai kritis. Hapus volume ES versi lama begitu pindah versi ES.

## Checklist

- [ ] Clone source 6.0.0 ke `app/` (`rm -rf app && git clone --branch 6.0.0 --depth 1 ...`)
- [ ] Cek `tail -6 app/Gemfile.lock` untuk versi Ruby/Bundler
- [ ] Update `Dockerfile` base image ke Ruby yang sesuai
- [ ] Cek apakah ES perlu naik versi lagi
- [ ] Build & fix gem issues (kalau ada) — test di container sementara dulu
- [ ] `rake db:migrate`
- [ ] Cek & sesuaikan config nginx untuk `/cable` WebSocket
- [ ] Rebuild search index (cek nama task dulu, hapus index nyangkut kalau ada dari
      percobaan gagal sebelumnya)
- [ ] Verifikasi UI di `helpdesk.satu.solutions`
- [ ] Update NOTES.md di folder ini dengan temuan baru
- [ ] **Buat CHANGELOG.md** (environment diff, schema changes dari daftar migrasi, fitur
      highlight dari CHANGELOG resmi Zammad 6.0.0 — pola sama seperti
      [../hop-4.0-to-5.0/CHANGELOG.md](../hop-4.0-to-5.0/CHANGELOG.md))
- [ ] **Buat RUNBOOK.md** (langkah final bersih + estimasi waktu tiap tahap — pola sama
      seperti [../hop-4.0-to-5.0/RUNBOOK.md](../hop-4.0-to-5.0/RUNBOOK.md))
- [ ] Update [../DOWNTIME_ESTIMATE.md](../DOWNTIME_ESTIMATE.md) dengan angka hop ini
