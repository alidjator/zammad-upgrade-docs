# Runbook — Hop 3.4.0 → 4.0

Ini adalah **langkah final yang terbukti benar**, hasil saringan dari proses debugging
panjang di [NOTES.md](NOTES.md). Dipakai untuk: (a) mengulang hop ini dari nol kalau
staging perlu di-reset, atau (b) jadi template saat hop ini benar-benar dieksekusi ke
instance produksi asli nanti.

**Estimasi total waktu eksekusi:** ~4,8 jam (didominasi reindex ES ~4,6 jam). Durasi per
tahap tercantum inline di tiap langkah di bawah; rincian lengkap di
[../DOWNTIME_ESTIMATE.md](../DOWNTIME_ESTIMATE.md).

## Pre-flight

- [ ] Backup database produksi (`mysqldump`), **verifikasi integritasnya** (`gzip -t`
  atau setara — JANGAN lanjut kalau gagal), simpan di luar server kalau memungkinkan.
  Detail lengkap: [../BACKUP_RESTORE.md](../BACKUP_RESTORE.md)
- [ ] Konfirmasi disk tersedia minimal 20GB bebas sebelum mulai (lihat insiden disk di
  NOTES.md — build cache Docker gampang menumpuk banyak selama proses)
- [ ] Kalau ini eksekusi ke **produksi sungguhan** (bukan staging): jadwalkan maintenance
  window ~1 jam untuk tahap 1-6 (migrasi schema saja cepat, ~1,5 menit), TAPI reindex ES
  di tahap 7 butuh ~4,6 jam tambahan — pertimbangkan jalankan reindex di background
  setelah UI sudah bisa diakses kembali (pencarian tidak akurat sementara sampai reindex
  selesai, tapi tiket tetap bisa dibuka/diedit normal)

## Langkah eksekusi

**1. Siapkan source code versi target**
```bash
git clone --branch 4.0.0 --depth 1 https://github.com/zammad/zammad.git app
```

**2. Patch Gemfile & Gemfile.lock** (WAJIB untuk versi 4.0.0 spesifik — gem yang sudah
tidak bisa di-install dari rubygems.org lagi):
```bash
sed -i "/gem 'tcr', git:/d" app/Gemfile
sed -i 's/^    mimemagic (0.3.5)$/    mimemagic (0.3.10)/' app/Gemfile.lock
sed -i '/^    mimemagic (0.3.10)$/a\      nokogiri (~> 1)\n      rake' app/Gemfile.lock
```

**3. Build image** (Dockerfile final ada di folder ini — base `ruby:2.6.6-stretch`,
termasuk `assets:precompile` di runtime, bukan build time)
```bash
docker compose build
```

**4. Jalankan container**
```bash
docker compose up -d
```

**5. Verifikasi koneksi database**
```bash
docker compose exec zammad-app env RAILS_ENV=production bundle exec rails runner \
  'puts ActiveRecord::Base.connection.execute("SELECT COUNT(*) FROM tickets").to_a'
```

**6. Migrasi schema** (~1,5 menit, 33 migrasi)
```bash
docker compose exec zammad-app env RAILS_ENV=production bundle exec rake db:migrate
```

**7. Rebuild search index** (~4,6 jam — tahap paling lama)
```bash
docker compose exec zammad-app env RAILS_ENV=production bundle exec rake searchindex:rebuild
```

## Verifikasi akhir

- [ ] Buka UI, login, cek dashboard tampil normal
- [ ] Coba search tiket — hasil muncul dengan benar
- [ ] Cek Admin Panel → System → Version menunjukkan versi yang benar

## Kalau perlu mundur (rollback)

Pola umum ada di [../BACKUP_RESTORE.md](../BACKUP_RESTORE.md) § "Pola umum rollback
per hop upgrade". Untuk hop ini: titik baginya adalah **tahap 6 (migrate)** — sebelum
itu tinggal hapus container/image, setelah itu wajib restore dari backup pre-flight.
