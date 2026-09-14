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
| 6.0 → 7.0 | ~12 menit | belum terukur presisi ⚠️ | ~3,1 jam (11.201 detik) | **~3,3 jam** (perkiraan) |

**Catatan transparansi metode pengukuran** — angka reindex ES di tabel ini didapat
dengan cara yang **sama untuk keempat hop**: menjumlahkan baris "done in X seconds"
yang dicetak `Benchmark.realtime` bawaan rake task Zammad sendiri
(`lib/tasks/zammad/search_index_es.rake`), didominasi step `Ticket` dan `User`.
Rinciannya ada di NOTES.md masing-masing hop: 3.4.0→4.0 (`Ticket` 15.144s/~4,2 jam,
`User` 1.296s/~21,6 menit), 4.0→5.0 (`Ticket` 13.570s/~3,8 jam, `User` 1.244s/~20,7
menit), 5.0→6.0 (`Ticket` 18.260s/~5,1 jam, `User` 1.790s/~29,8 menit), 6.0→7.0
(`Ticket` 9.600s/~2,7 jam, `User` 1.404s/~23,4 menit). Untuk
**hop 6.0→7.0**, migrasi schema-nya sendiri belum diukur presisi (fokus saat eksekusi
ada di debugging bug urutan migrasi `recent_closes` — lihat
[hop-6.0-to-7.0/NOTES.md](hop-6.0-to-7.0/NOTES.md) Insiden 6) — 78 migrasi historis
kemungkinan besar di bawah 3 menit total berdasarkan durasi tiap migrasi individual
yang sempat terlihat (mayoritas <1 detik, beberapa migrasi berat individual belasan
detik), tapi ini perkiraan, bukan angka terukur.

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
