# Estimasi Downtime — Referensi untuk Perencanaan Maintenance Window

Angka di bawah adalah **hasil pengukuran nyata** dari eksekusi di staging (bukan
perkiraan teoritis), jadi cukup representatif untuk perencanaan window produksi —
dengan catatan: performa staging server ini juga berbagi resource dengan banyak
service lain (OCR, CI runner, dll), jadi di server produksi yang lebih lega,
kemungkinan bisa lebih cepat dari angka ini.

## Ringkasan per hop

| Hop | Build image | Migrasi schema | Reindex ES | Total (kalau reindex ditunggu) |
|---|---|---|---|---|
| 3.4.0 → 4.0 | ~8 menit | 1m33s | ~4,6 jam | **~4,8 jam** |
| 4.0 → 5.0 | ~8 menit | 1m31s | ~4,3 jam | **~4,5 jam** |
| 5.0 → 6.0 | ~11 menit | 9m25s | ~5,6 jam | **~5,9 jam** |
| 6.0 → 7.0 | ~12 menit | 6m11s (151 migrasi) | ~3,1 jam (11.201 detik) | **~3,3 jam** |

**Catatan transparansi metode pengukuran** — angka reindex ES di tabel ini didapat
dengan cara yang **sama untuk keempat hop**: menjumlahkan baris "done in X seconds"
yang dicetak `Benchmark.realtime` bawaan rake task Zammad sendiri
(`lib/tasks/zammad/search_index_es.rake`), didominasi step `Ticket` dan `User`.
Rinciannya ada di NOTES.md masing-masing hop: 3.4.0→4.0 (`Ticket` 15.144s/~4,2 jam,
`User` 1.296s/~21,6 menit), 4.0→5.0 (`Ticket` 13.570s/~3,8 jam, `User` 1.244s/~20,7
menit), 5.0→6.0 (`Ticket` 18.260s/~5,1 jam, `User` 1.790s/~29,8 menit), 6.0→7.0
(`Ticket` 9.600s/~2,7 jam, `User` 1.404s/~23,4 menit).

**Migrasi schema hop 6.0→7.0** juga terukur presisi lewat sumber lain — timestamp
`Rails.logger` di `log/production.log` (baris "Migrating to X"), bukan dari
`Benchmark.realtime` per-step seperti reindex. Migrasi pertama
(`SettingAddStoreProviderS3`) tercatat `08:17:08`, migrasi terakhir
(`Pr5952FixTypos`) `08:23:19` → total **6 menit 11 detik untuk 151 migrasi** (bukan 78
seperti dugaan awal — lihat [hop-6.0-to-7.0/CHANGELOG.md](hop-6.0-to-7.0/CHANGELOG.md)
untuk penjelasan kenapa hitungan awal keliru). Jeda diagnosis manual Insiden 6
(bug urutan migrasi `recent_closes`) cuma menyumbang 38 detik dari total ini — tidak
signifikan menambah durasi.

## Standardisasi metode pengukuran (mulai hop 7.0→7.1.3)

Empat hop di atas diukur dengan **3 cara berbeda** yang kebetulan menghasilkan angka
akurat, tapi tidak seragam prosesnya:
- Reindex ES (semua hop): baca baris "done in X seconds" dari `Benchmark.realtime`
  bawaan rake task Zammad sendiri.
- Migrasi schema hop 6.0→7.0: hitung selisih timestamp `Rails.logger` di
  `log/production.log`.
- Migrasi schema hop 1-3: metode tidak terdokumentasikan (angka `1m33s`/`1m31s`/`9m25s`
  ada di NOTES.md/RUNBOOK.md tapi caranya diukur tidak pernah ditulis eksplisit —
  kemungkinan observasi manual atau output verbose Rails, tidak bisa diverifikasi ulang
  sekarang karena hop-nya sudah selesai).

**Mulai hop 7.0→7.1.3 (dan setiap re-eksekusi hop manapun setelah ini), gunakan `time`
untuk SEMUA tahap berwaktu** (build, migrate, reindex) — satu cara yang sama, tidak
bergantung pada apakah aplikasi kebetulan mencatat timestamp yang bisa ditelusuri:
```bash
time docker compose build zammad-app
time docker compose exec zammad-app env RAILS_ENV=production bundle exec rake db:migrate
time docker compose exec zammad-app env RAILS_ENV=production bundle exec rake zammad:searchindex:rebuild
```
Catat angka `real` (wall-clock, bukan `user`/`sys`) dari output `time` sebagai angka
resmi di RUNBOOK.md hop tersebut. Kalau proses dijalankan di dalam `screen` dan
sempat di-detach, `time` tetap mencetak hasilnya ke layar begitu command selesai —
tinggal `screen -r` atau `hardcopy` sebelum layar tertimpa command lain.

## Yang PALING menentukan durasi: reindex Elasticsearch

Di ketiga hop yang sudah selesai, reindex ES adalah **>90% dari total waktu**, didominasi
satu tabel: `tickets` (161.884 baris → ~3,8-5,1 jam sendirian, cenderung naik tiap hop
karena beban server bersama yang bertambah). Migrasi schema database biasanya sangat
cepat (di bawah 2 menit) — KECUALI kalau satu hop mencakup banyak rilis minor sekaligus
(hop 5.0→6.0 mencakup 5 rilis minor yang dilewati, migrasinya jadi 9m25s karena jumlah
migrasi jauh lebih banyak, termasuk 2 migrasi berat individual >2 menit).

**Implikasi penting untuk perencanaan produksi:** UI Zammad (login, buka/edit tiket)
sudah bisa dipakai normal **begitu migrasi schema selesai** (~2 menit) — TIDAK perlu
menunggu reindex ES selesai untuk membuka akses ke user. Yang belum akurat selama
reindex berjalan cuma **fitur pencarian tiket** (search bisa menunjukkan hasil
tidak lengkap/kosong sampai reindex tuntas).

**Rekomendasi strategi maintenance window:**
1. Window "keras" (user benar-benar tidak bisa akses): cuma untuk tahap build + migrate
   → **~10 menit** per hop
2. Reindex ES (~4-5 jam) bisa dijalankan **setelah** akses dibuka kembali, sebagai proses
   background — informasikan ke user bahwa pencarian tiket mungkin belum akurat 100%
   sampai beberapa jam ke depan

## Faktor lain yang menambah waktu (di luar angka tabel)

- **Restore database** (kalau harus restore ke database baru, misal saat migrasi ke
  MariaDB legacy di hop 4.0→5.0): ~18-20 menit untuk ~800MB dump terkompresi
  (161rb tiket, 1 juta+ ticket_article)
- **Disk cleanup** (`docker builder prune`, dll) — sebaiknya dilakukan **sebelum** hop
  mulai, bukan dihitung sebagai bagian downtime, tapi perlu dialokasikan waktu terpisah
  di jadwal kalau disk sudah mepet
- **Retry reindex akibat insiden** (hop 6.0→7.0) — angka ~3,1 jam di tabel di atas
  cuma durasi PROSES BERSIH (percobaan yang berhasil). Total wall-clock sungguhan
  jauh lebih lama karena 2 percobaan gagal sebelumnya (index stale, lalu ES masuk mode
  read-only karena disk penuh — lihat NOTES.md Insiden 8 & 9) yang masing-masing perlu
  diagnosis manual sebelum retry. Untuk perencanaan produksi, alokasikan buffer waktu
  ekstra di luar angka "bersih" ini kalau kondisi disk server produksi belum dipastikan
  lega jauh di bawah 90% sebelum reindex dimulai.

## Update dokumen ini

Tambahkan baris baru ke tabel di atas setiap hop selesai — datanya sudah tercatat di
`hop-X-to-Y/RUNBOOK.md` masing-masing, tinggal disalin ringkasannya ke sini.
