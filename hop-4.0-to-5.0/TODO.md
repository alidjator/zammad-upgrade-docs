# Hop 4.0 → 5.0 — Belum Dimulai

## Requirement (lihat [../ROADMAP.md](../ROADMAP.md) untuk detail lengkap)

- Ruby: 2.6.6 → **2.7.4**
- Elasticsearch: **wajib naik ke ≥7.8, <8** sebelum hop ini (ES saat ini masih 6.8.23 dari hop
  sebelumnya — ini beda dari hop 3.4→4.0 yang tidak perlu ganti ES)
- Database: Postgres ≥9.3 / MySQL ≥5.5.8 (MariaDB 11.8.3 kita sudah jauh di atas ini, aman)
- Node.js jadi wajib untuk `rake assets:precompile` (sudah ada dari hop sebelumnya, tapi
  sekarang jadi hard requirement resmi, bukan cuma kebiasaan)

## Langkah yang sudah terbukti perlu diulang tiap hop (dari NOTES.md hop sebelumnya)

1. `git clone --branch 5.0.0 --depth 1 https://github.com/zammad/zammad.git app`
2. Cek `tail -5 app/Gemfile.lock` untuk versi Ruby & Bundler yang cocok
3. Cek gem yang mungkin sudah yanked/dead git ref — coba `bundle install` di container
   sementara dulu (pola sama seperti [hop-3.4.0-to-4.0/NOTES.md](../hop-3.4.0-to-4.0/NOTES.md) #4-#6)
   sebelum masuk ke Dockerfile beneran
4. **Upgrade Dockerfile.elasticsearch ke base image ES ≥7.8** — ini WAJIB baru di hop ini,
   belum pernah dilakukan sebelumnya. Cek juga apakah plugin `ingest-attachment` masih
   dibutuhkan/kompatibel di ES 7.x.
5. Setelah build, jangan lupa `rake assets:precompile` di Dockerfile (sudah jadi kebiasaan
   permanen sejak hop sebelumnya)
6. Cek nama rake task search index lagi (`rake --tasks | grep -i -E "index|search"`) —
   sudah terbukti berubah-ubah antar versi
7. Test di staging (database `zammad_staging`, port 3010/16043) sebelum declare selesai

## Belum dikerjakan — checklist

- [ ] Clone source 5.0.0 ke `app/`
- [ ] Upgrade `Dockerfile.elasticsearch` ke ES 7.8+
- [ ] Update `Dockerfile` base image ke Ruby 2.7.4
- [ ] Build & fix gem issues (kalau ada)
- [ ] `rake db:migrate`
- [ ] `rake assets:precompile` (pastikan sudah masuk Dockerfile)
- [ ] Rebuild search index (cek nama task dulu)
- [ ] Verifikasi UI di `helpdesk.satu.solutions`
- [ ] Update NOTES.md di folder ini dengan temuan baru
