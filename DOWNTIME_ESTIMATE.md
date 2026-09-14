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

## Update dokumen ini

Tambahkan baris baru ke tabel di atas setiap hop selesai — datanya sudah tercatat di
`hop-X-to-Y/RUNBOOK.md` masing-masing, tinggal disalin ringkasannya ke sini.
