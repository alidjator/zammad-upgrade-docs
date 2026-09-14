# Migrasi MariaDB → PostgreSQL — Belum Dimulai

**Status prasyarat: TERPENUHI.** Zammad sekarang di versi 6.0.0 (≥5.3 minimum untuk tool
migrasi resmi `rake zammad:db:pgloader`). Bisa dimulai kapan saja.

## Kenapa ini wajib

Mulai **Zammad 7.0, MySQL/MariaDB dihapus total** dari dukungan — PostgreSQL jadi
satu-satunya opsi. Migrasi ini harus selesai SEBELUM hop 6.0 → 7.0.

## Rencana yang sudah dibahas sebelumnya (ringkasan)

Sumber data: MariaDB 10.11 (`zammad-mariadb-legacy`, database `zammad_staging`) — bukan
lagi MariaDB 11.8.3 produksi (itu tidak disentuh sejak awal proyek ini).

### Langkah garis besar

1. Tambah service `zammad-postgres` (image `postgres:15` atau lebih baru) ke
   `docker-compose.yml`
2. Generate command file: `docker compose exec zammad-app env RAILS_ENV=production bundle exec rake zammad:db:pgloader > pgloader-command`
3. Edit command file: pastikan source MariaDB (`zammad-mariadb-legacy`) dan target
   Postgres (`zammad-postgres`) sudah benar
4. `database.yml` diubah ke adapter `postgresql`, arahkan ke `zammad-postgres`
5. `rake db:create` untuk buat database kosong di Postgres
6. Jalankan `pgloader` (container terpisah `dimitri/pgloader`, network sama) dengan
   command file dari langkah 2
7. Validasi row count tiap tabel penting (sama seperti validasi restore MariaDB
   sebelumnya) sebelum lanjut

### Hal yang WAJIB dicek ulang sebelum eksekusi (sudah berubah beberapa kali di hop-hop sebelumnya)

- **Cek isi command file hasil rake task** — jangan asumsikan formatnya sama seperti
  contoh di dokumentasi resmi, task ini bisa saja berubah generate command yang berbeda
  di versi 6.0.0
- **Cek privilege user MariaDB source** — restore/dump sebelumnya sempat kena masalah
  `DEFINER` butuh SUPER privilege (lihat hop-3.4.0-to-4.0/NOTES.md #masalah 7) — pgloader
  membaca langsung dari MariaDB, mungkin butuh privilege serupa
- **Backup MariaDB legacy dulu** sebelum migrasi (walau ini "cuma" staging, proses hop
  berikutnya bergantung penuh pada data ini)

## Checklist

- [ ] Backup database `zammad_staging` di MariaDB legacy
- [ ] Tambah service `zammad-postgres` ke docker-compose.yml
- [ ] Generate & edit command file pgloader
- [ ] Jalankan migrasi via pgloader
- [ ] Validasi row count semua tabel utama (tickets, ticket_articles, users, organizations)
- [ ] Update `database.yml` ke adapter postgresql
- [ ] Rebuild image (Gemfile.lock kemungkinan sudah punya gem `pg`, tidak perlu tambahan)
- [ ] Jalankan aplikasi, verifikasi UI & search normal
- [ ] Update NOTES.md/CHANGELOG.md/RUNBOOK.md di folder ini
- [ ] Update README.md status
