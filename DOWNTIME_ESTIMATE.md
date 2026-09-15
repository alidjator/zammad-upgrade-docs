# Estimasi Downtime — Referensi untuk Perencanaan Maintenance Window

Angka di bawah adalah **hasil pengukuran nyata** dari eksekusi di sandbox riset
(bukan perkiraan teoritis), sehingga cukup representatif untuk perencanaan window
produksi nyata nanti — dengan 2 catatan penting:
1. Performa sandbox ini berbagi resource server dengan banyak service lain (OCR, CI
   runner, dll), sehingga di server produksi yang lebih lega, kemungkinan bisa lebih
   cepat dari angka ini.
2. **Data yang diuji adalah snapshot produksi yang diambil di masa lalu** (lihat
   [README.md § Konteks](README.md#konteks)), bukan volume data produksi yang berjalan
   saat ini — durasi reindex ES terbukti berbanding lurus dengan jumlah tiket, sehingga
   jika data produksi nyata sudah jauh lebih banyak, **skalakan ulang angka di bawah**,
   jangan pakai langsung apa adanya. Lihat [CUTOVER_CHECKLIST.md § 0. Sebelum
   menjadwalkan tanggal cutover](CUTOVER_CHECKLIST.md#0-sebelum-menjadwalkan-tanggal-cutover).

## Ringkasan per hop

| Hop | Build image | Migrasi schema | Reindex ES | Total (jika reindex ditunggu) |
|---|---|---|---|---|
| 3.4.0 → 4.0 | 32m17s | 1m33s | ~4,6 jam | **~5,2 jam** |
| 4.0 → 5.0 | 62m2s | 1m31s | ~4,3 jam | **~5,4 jam** |
| 5.0 → 6.0 | 12m0s | 9m25s | ~5,6 jam | **~6,0 jam** |
| 6.0 → 7.0 | 12m7s | 6m11s | ~3,1 jam | **~3,4 jam** |
| 7.0 → 7.1.3 | 4m36s | 4s | tidak perlu | **~4m40s** |

*(Format konsisten: durasi utama saja di tiap sel. Rincian tambahan — jumlah migrasi,
breakdown detik per model, dll — ada di catatan prosa di bawah, bukan di dalam tabel.)*

**Hop 7.0 → 7.1.3 adalah hop TERAKHIR proyek ini** — jauh lebih ringan dari semua hop
sebelumnya: tidak perlu `searchindex:rebuild` sama sekali (tidak ada perubahan skema
index di `BREAKING_CHANGES.md` 7.1), migrasi cuma 20 migrasi/4 detik, build ~4,5 menit.
Total downtime keras (build+migrate) untuk hop ini **di bawah 5 menit** — kontras
tajam dengan hop 4.0→5.0 yang butuh lebih dari 1 jam untuk tahap yang sama.

⚠️ **Angka Build image untuk hop 3.4.0→4.0 dan 4.0→5.0 dikoreksi tanggal 15 Sept 2026** —
sebelumnya tertulis "~8 menit" untuk keduanya, tidak berdasarkan sumber terverifikasi
apa pun (lihat bagian "Standardisasi metode pengukuran" di bawah). Angka baru didapat
dengan menelusuri **log daemon Docker (`journalctl -u docker`)**, yang ternyata masih
tersimpan mundur sampai 15 hari (reboot harian server, tapi journal persisten lintas
boot) — jauh melebihi ekspektasi awal. Metodenya: cari timestamp `"image pulled"`
untuk base image tiap hop (`ruby:2.6.6-stretch`, `ruby:2.7.4-buster`, dst.), lalu cari
timestamp `sbJoin` (container pertama terhubung ke network) untuk container
`zammad-staging-zammad-app-1` setelahnya — selisihnya adalah durasi build+up total.
Hop 5.0→6.0 dan 6.0→7.0 ternyata SUDAH dekat dengan angka lama (12m0s vs ~11 menit,
12m7s vs ~12 menit) — cuma hop 1 dan 2 yang meleset jauh (4-8x lipat).

**Temuan penting dari penelusuran ini:** log menunjukkan hop 3.4.0→4.0 (traceID sama
untuk 3 span "exporting to image") dan hop 4.0→5.0 (3 traceID BERBEDA, masing-masing
~12-17 menit terpisah) **SAMA-SAMA kena masalah "build 3 image terpisah"** yang baru
disadari dan diperbaiki di hop 6.0→7.0 (lihat [hop-6.0-to-7.0/NOTES.md](hop-6.0-to-7.0/NOTES.md)
Insiden 4) — bukan masalah baru khusus hop terakhir, tapi sudah ada sejak hop pertama.
Hop 5.0→6.0 kelihatan sudah lebih efisien (beberapa span "exporting to image" berbagi
traceID yang sama, tanda cache Docker terpakai across service), menjelaskan kenapa
durasinya jauh lebih pendek dari hop 1-2 meski base image-nya lebih besar/baru.

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

**Jumlah migrasi per hop** (jika tercatat): 3.4.0→4.0 — **33 migrasi**
([hop-3.4.0-to-4.0/RUNBOOK.md](hop-3.4.0-to-4.0/RUNBOOK.md)); 4.0→5.0 dan 5.0→6.0 —
tidak pernah dicatat jumlahnya di dokumentasi manapun; 6.0→7.0 — **151 migrasi**
(lihat di atas).

**Migrasi schema hop 1-3 (`1m33s`/`1m31s`/`9m25s`) TERBUKTI tidak bisa ditelusuri
ulang** — dicek 15 Sept 2026 lewat `docker ps -a` dan `/var/lib/docker/containers/`,
cuma ada 1 container `zammad-app` yang tersisa (dibuat 14 Sept, punya hop 6.0→7.0 ini).
Beda dari waktu build (tercatat di *daemon* Docker level host, bertahan lintas
penggantian container) atau migrasi hop 6.0→7.0 (tercatat di `log/production.log`,
kebetulan container-nya masih hidup saat ditelusuri), log migrasi hop 1-3 tersimpan di
filesystem container HOP ITU SENDIRI yang sudah lama diganti/dihapus oleh build hop
berikutnya (`docker compose up -d` untuk image baru otomatis stop+hapus container
lama beserta seluruh filesystem-nya, termasuk `log/`, karena tidak ada satu pun
docker-compose.yml yang me-mount `log/` ke volume persisten). Ini kesimpulan final,
bukan sekadar "belum ketemu" — sumber datanya memang sudah terbukti tidak ada lagi.

## Standardisasi metode pengukuran (mulai hop 7.0→7.1.3)

Empat hop di atas diukur dengan **3 cara berbeda** yang kebetulan menghasilkan angka
akurat, tapi tidak seragam prosesnya:
- Reindex ES (semua hop): baca baris "done in X seconds" dari `Benchmark.realtime`
  bawaan rake task Zammad sendiri.
- Migrasi schema hop 6.0→7.0: hitung selisih timestamp `Rails.logger` di
  `log/production.log`.
- Migrasi schema hop 1-3: metode tidak terdokumentasikan, dan **terbukti tidak bisa
  ditelusuri ulang lagi** — sumbernya (`log/production.log` di container hop tersebut)
  sudah dihapus permanen sejak container itu diganti oleh build hop berikutnya (detail
  di atas).

**Keputusan: TIDAK pakai `time`.** `time` cuma mencetak hasilnya SEKALI ke layar begitu
command selesai — jika sesi `screen`-nya sempat tertimpa command lain sebelum sempat
dibaca (persis yang terjadi ke `1142032.hop7-reindex` di hop ini), angkanya hilang
selamanya dan tidak bisa direkonstruksi lagi. Bandingkan dengan migrasi hop 6.0→7.0 di
atas — datanya masih bisa diselamatkan justru karena Rails **sendiri** sudah menulis
timestamp ke file log (`log/production.log`), bukan cuma ke layar.

**Mulai hop 7.0→7.1.3, prinsipnya: manfaatkan fitur pelaporan bawaan tiap command, dan
pastikan outputnya disimpan ke FILE (bukan cuma layar) supaya tahan terhadap sesi
`screen` yang tertimpa/terputus:**

```bash
# Build — Docker Buildx sudah mencetak total durasi di baris ringkasannya sendiri
# ("[+] Building 727.0s (13/13) FINISHED"), tinggal disimpan ke file dengan tee.
docker compose build zammad-app 2>&1 | tee build-hop7.1.3.log

# Migrate — TIDAK perlu redirect tambahan sama sekali. Rails.logger otomatis dan
# selalu menulis tiap "Migrating to X" ke log/production.log secara persisten,
# terlepas dari sesi terminal/screen apa pun. Ambil durasi kapan saja setelahnya:
docker compose exec zammad-app env RAILS_ENV=production bundle exec rake db:migrate
docker compose exec zammad-app grep "Migrating to" log/production.log | head -1
docker compose exec zammad-app grep "Migrating to" log/production.log | tail -1

# Reindex — "done in X seconds" dari Benchmark.realtime cuma cetak ke STDOUT, TIDAK
# otomatis masuk ke file log manapun. WAJIB di-tee supaya tidak bergantung buffer
# screen yang terbatas.
docker compose exec zammad-app env RAILS_ENV=production bundle exec rake zammad:searchindex:rebuild 2>&1 | tee reindex-hop7.1.3.log
```

Tetap jalankan ketiganya di dalam `screen` seperti biasa (untuk ketahanan terhadap
koneksi SSH terputus), tapi jangan lagi bergantung pada buffer `screen` sebagai
satu-satunya tempat data durasi tersimpan — `tee`/log Rails yang jadi sumber utama.

## Yang PALING menentukan durasi: reindex Elasticsearch

Di keempat hop, reindex ES tetap **porsi terbesar dari total waktu (~80-93%,
bervariasi per hop)**, didominasi satu tabel: `tickets` (161.894 baris → ~2,7-5,1 jam
sendirian, sempat naik tiap hop karena beban server bersama yang bertambah, lalu turun
lagi di hop 6.0→7.0 karena beban server saat itu lebih ringan). Migrasi schema
database biasanya cepat (di bawah 10 menit) — KECUALI jika satu hop mencakup banyak
rilis minor sekaligus (hop 5.0→6.0 dan 6.0→7.0 sama-sama melompati banyak rilis minor,
migrasinya jadi 9m25s dan 6m11s karena jumlah migrasi jauh lebih banyak).

**Build image TIDAK selalu kecil** — koreksi setelah penelusuran journalctl (lihat
tabel di atas): hop 4.0→5.0 build-nya sendiri makan **~1 jam 2 menit**, lebih lama dari
migrasi schema hop manapun. Jangan asumsikan build cuma "beberapa menit" saat
merencanakan window — cek dulu apakah compose-nya sudah pakai `image:` yang sama untuk
app/websocket/scheduler (lihat [hop-6.0-to-7.0/RUNBOOK.md](hop-6.0-to-7.0/RUNBOOK.md)),
Jika belum, build bisa 3x lebih lama dari seharusnya.

**Implikasi penting untuk perencanaan produksi:** UI Zammad (login, buka/edit tiket)
sudah bisa dipakai normal **begitu migrasi schema selesai** (~2 menit) — TIDAK perlu
menunggu reindex ES selesai untuk membuka akses ke user. Yang belum akurat selama
reindex berjalan cuma **fitur pencarian tiket** (search bisa menunjukkan hasil
tidak lengkap/kosong sampai reindex tuntas).

**Rekomendasi strategi maintenance window:**
1. Window "keras" (user benar-benar tidak bisa akses): untuk tahap build + migrate
   → **berkisar 18-64 menit per hop** (bukan "~10 menit" seperti perkiraan sebelumnya
   yang ternyata tidak berdasar — lihat koreksi Build image di atas). Pastikan
   `docker compose build` sudah dites dulu di staging untuk hop yang sama sebelum
   menetapkan angka window produksi, jangan asumsikan cepat.
2. Reindex ES (~3-6 jam) bisa dijalankan **setelah** akses dibuka kembali, sebagai proses
   background — informasikan ke user bahwa pencarian tiket mungkin belum akurat 100%
   sampai beberapa jam ke depan

## Faktor lain yang menambah waktu (di luar angka tabel)

- **Restore database** (jika harus restore ke database baru, misal saat migrasi ke
  MariaDB legacy di hop 4.0→5.0): ~18-20 menit untuk ~800MB dump terkompresi
  (161rb tiket, 1 juta+ ticket_article)
- **Disk cleanup** (`docker builder prune`, dll) — sebaiknya dilakukan **sebelum** hop
  mulai, bukan dihitung sebagai bagian downtime, tapi perlu dialokasikan waktu terpisah
  di jadwal jika disk sudah mepet
- **Retry reindex akibat insiden** (hop 6.0→7.0) — angka ~3,1 jam di tabel di atas
  cuma durasi PROSES BERSIH (percobaan yang berhasil). Total wall-clock sungguhan
  jauh lebih lama karena 2 percobaan gagal sebelumnya (index stale, lalu ES masuk mode
  read-only karena disk penuh — lihat NOTES.md Insiden 8 & 9) yang masing-masing perlu
  diagnosis manual sebelum retry. Untuk perencanaan produksi, alokasikan buffer waktu
  ekstra di luar angka "bersih" ini jika kondisi disk server produksi belum dipastikan
  lega jauh di bawah 90% sebelum reindex dimulai.

## Update dokumen ini

Tambahkan baris baru ke tabel di atas setiap hop selesai — datanya sudah tercatat di
`hop-X-to-Y/RUNBOOK.md` masing-masing, tinggal disalin ringkasannya ke sini.
