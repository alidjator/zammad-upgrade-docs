# Migrasi MariaDB → PostgreSQL — Catatan Lengkap (Status: ✅ Selesai & Tervalidasi)

## Ringkasan

Migrasi database staging dari MariaDB 10.11 (`zammad-mariadb-legacy`) ke PostgreSQL 16.13
(sudah berjalan di host, dipakai bersama aplikasi lain dengan traffic rendah). Dilakukan
saat Zammad sudah di versi 6.0.0 (memenuhi syarat minimum ≥5.3 untuk tool resmi
`rake zammad:db:pgloader`).

**Hasil: 0 error di seluruh proses, row count semua tabel utama cocok 100%.**

## Keputusan arsitektur

- **PostgreSQL dipakai dari instance yang SUDAH ADA di host** (bukan container baru),
  atas permintaan user — instance ini dipakai bersama aplikasi lain, traffic rendah.
- **Isolasi:** dibuat user & database baru khusus (`zammad_staging_pg`), TIDAK reuse
  database/role aplikasi lain di instance yang sama. Privilege dibatasi ke database ini saja.
- Koneksi dari container ke PostgreSQL host: `host.docker.internal` (perlu `extra_hosts:
  host-gateway` di docker-compose.yml — sempat dihapus saat pindah ke MariaDB container,
  ditambahkan lagi khusus untuk ini).

## Prasyarat yang dicek dulu

1. **Versi PostgreSQL:** 16.13 — jauh di atas minimum Zammad (≥13), tidak ada kekhawatiran
   versi-gap seperti kasus MariaDB 11.x sebelumnya (PostgreSQL historisnya jauh lebih
   stabil kompatibilitas wire-protocol/adapter-nya dibanding MySQL family).
2. **`listen_addresses = *`** — PostgreSQL sudah dengar di semua interface, tidak perlu diubah.
3. **`pg_hba.conf`** — sudah ada rule `host all all 0.0.0.0/0 md5` (mengizinkan password
   auth dari IP manapun) — cukup longgar tapi sudah ada sebelumnya, tidak diubah.
4. **Firewall (firewalld)** — rule yang ada cuma untuk interface `eth0` (publik), traffic
   dari Docker bridge ke host tidak terpengaruh. Tidak perlu penyesuaian.

## Langkah eksekusi (ringkasan — detail di RUNBOOK.md)

1. Buat user + database terisolasi di PostgreSQL host
2. Backup MariaDB legacy dulu (`mariadb-dump`) — **sempat gagal di percobaan pertama**
   karena proses laptop pengguna terputus di tengah jalan, menghasilkan file gzip
   corrupt (`gzip -t` gagal dengan "unexpected end of file"). Pelajaran: **selalu
   verifikasi integritas backup** (`gzip -t`) sebelum melanjutkan, jangan asumsikan
   backup selesai hanya karena command "selesai" tanpa terlihat errornya.
3. Generate command pgloader: `rake zammad:db:pgloader` — otomatis mendeteksi koneksi
   source MariaDB dari `database.yml` yang aktif, tinggal isi target PostgreSQL manual
4. Jalankan pgloader via container terpisah (`dimitri/pgloader`), join network Docker
   Compose (`zammad-staging_default`) + `--add-host host.docker.internal:host-gateway`
5. **Proses memakan waktu lama (43 menit) — dijalankan di dalam `screen`** supaya tidak
   terputus jika koneksi SSH/laptop pengguna bermasalah lagi (permintaan eksplisit user
   setelah insiden backup terputus di atas)
6. Validasi row count MariaDB vs PostgreSQL untuk tabel utama
7. Ubah `database.yml` ke adapter `postgresql`, restart container
8. Verifikasi `ActiveRecord::Base.connection.adapter_name` = "PostgreSQL", cek
   `db:migrate:status` (semua "up", tidak ada yang pending), verifikasi UI

## Hasil pgloader (dari log lengkap)

- Total waktu: **43m41s**
- Total baris: **11.375.352** (3,1GB)
- **0 error** di semua tahap: COPY Threads (4), Create Indexes (504), Index Build (504),
  Reset Sequences (98), Primary Keys (100), Create Foreign Keys (178)

## Validasi row count (MariaDB vs PostgreSQL)

| Tabel | MariaDB | PostgreSQL | Match |
|---|---|---|---|
| tickets | 161.884 | 161.884 | ✅ |
| ticket_articles | 1.031.884 | 1.031.884 | ✅ |
| users | 72.010 | 72.010 | ✅ |
| organizations | 1.851 | 1.851 | ✅ |

## Yang TIDAK perlu dilakukan ulang

- **Rebuild search index ES** — tidak diperlukan. Data tiket/user/dsb tidak berubah isi
  atau strukturnya secara logis (cuma pindah backend database), jadi index ES yang sudah
  ada dari hop 5.0→6.0 masih valid dan akurat.
- **Re-run migrasi Rails** — `db:migrate:status` menunjukkan semua migrasi sudah `up`,
  pgloader ikut memindahkan tabel `schema_migrations`, jadi status migrasi terbawa utuh.

## Status MariaDB legacy setelah migrasi

`zammad-mariadb-legacy` **belum dihapus** — dibiarkan jalan sebagai jaring pengaman
rollback untuk sementara. Rencana: matikan (bukan hapus volume) setelah beberapa hari
stabil di PostgreSQL, baru pertimbangkan hapus total untuk membebaskan disk.
