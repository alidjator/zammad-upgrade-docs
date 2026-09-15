# Backup & Restore — Referensi Darurat

Satu tempat rujukan cepat untuk backup/restore, dipisah dari RUNBOOK per hop supaya
tidak perlu menelusuri hop mana dulu kalau kondisi darurat butuh restore cepat. Setiap
`RUNBOOK.md` per hop cross-reference ke sini untuk langkah backup pre-flight-nya.

**Aturan mutlak, berlaku untuk SEMUA backup di proyek ini (staging maupun produksi
nyata nanti):** setiap backup WAJIB diverifikasi integritasnya sebelum dianggap valid
— jangan asumsikan sukses hanya karena command selesai tanpa error terlihat. Ini bukan
teori: pernah terjadi insiden nyata di proyek ini (lihat § Insiden nyata di bawah)
di mana koneksi terputus di tengah proses menghasilkan file backup yang TERLIHAT ada
tapi isinya korup/terpotong.

## Backup — database saat ini di sandbox (PostgreSQL)

```bash
# Ganti <db> dan <user> sesuai database.yml aktif di server
pg_dump -h host.docker.internal -U <user> <db> | gzip > backup_$(date +%Y%m%d_%H%M%S).sql.gz

# WAJIB: verifikasi integritas sebelum lanjut apa pun
gzip -t backup_*.sql.gz && echo "GZIP OK — aman dilanjutkan" || echo "GAGAL — ulangi backup, JANGAN lanjut"
```

## Backup — database MariaDB (kalau masih relevan, mis. `zammad-mariadb-legacy`)

```bash
docker compose exec <service_mariadb> mariadb-dump -u root -p'<root_password>' <database> | gzip > backup_$(date +%Y%m%d_%H%M%S).sql.gz
gzip -t backup_*.sql.gz && echo "GZIP OK" || echo "GAGAL — ulangi"
```

**Catatan histori penting:** binary yang benar di MariaDB 10.6+/11.x adalah
`mariadb`/`mariadb-dump`, BUKAN `mysql`/`mysqldump` (nama lama tidak selalu tersedia
lagi di image MariaDB modern).

## Restore — dari backup PostgreSQL

```bash
gunzip -c backup_YYYYMMDD_HHMMSS.sql.gz | psql -h host.docker.internal -U <user> <db_target>
```

**Sebelum restore ke database yang sedang dipakai container aktif:** stop dulu
container yang connect ke database itu (`docker compose stop zammad-app
zammad-websocket zammad-scheduler`) supaya tidak ada koneksi aktif yang mengganggu
proses restore atau menulis data baru di tengah proses.

## Restore — dari backup MariaDB

```bash
gunzip -c backup_YYYYMMDD_HHMMSS.sql.gz | docker compose exec -T <service_mariadb> mariadb -u root -p'<root_password>' <database>
```

**Restore WAJIB pakai user `root`, bukan user aplikasi terbatas** — dump Zammad berisi
definisi trigger dengan `DEFINER=`crontab`@`%`\`` yang butuh privilege SUPER (pelajaran
dari hop 4.0→5.0).

## Insiden nyata — kenapa verifikasi integritas ini bukan formalitas

Saat migrasi MariaDB→PostgreSQL, proses `mariadb-dump | gzip` sempat terputus di
tengah jalan karena gangguan koneksi laptop pelaksana. Command "selesai" tanpa pesan
error yang jelas terlihat, tapi file hasilnya ternyata korup — `gzip -t` gagal dengan
"unexpected end of file", dan isinya berhenti di tengah satu pernyataan `INSERT`.
Diperbaiki dengan mengulang proses sampai tuntas dan mem-verifikasi ulang (728MB,
valid). Detail lengkap di [postgres-migration/NOTES.md](postgres-migration/NOTES.md).
**Pelajaran:** selalu verifikasi, jangan asumsikan selesai = valid — terutama untuk
proses panjang yang rentan terputus (jalankan di dalam `screen` untuk backup besar).

## Pola umum rollback per hop upgrade

Semua hop upgrade (3.4.0→4.0, 4.0→5.0, 5.0→6.0, dst.) punya pola rollback yang SAMA —
RUNBOOK per hop cross-reference ke sini, cuma sebut detail spesifik hop (nomor tahap
migrate, nama container/database yang relevan):

- **Sebelum tahap migrate schema dijalankan**: rollback selalu aman & sederhana —
  cukup hapus container/image hop yang sedang dikerjakan. Database/versi sebelumnya
  TIDAK tersentuh sama sekali (build & boot container baru tidak mengubah data apa
  pun sampai `db:migrate` benar-benar dijalankan).
- **Setelah tahap migrate schema dijalankan**: rollback TIDAK sesederhana itu lagi —
  migrasi Rails tidak didesain untuk di-reverse otomatis. **Satu-satunya jalan mundur
  yang aman adalah restore dari backup pre-flight** (§ Restore di atas), bukan
  mencoba downgrade schema manual.

## Kalau butuh restore SEKARANG (kondisi darurat)

1. Cari backup TERAKHIR yang lolos verifikasi `gzip -t` — jangan pakai backup yang
   belum diverifikasi meski tampak paling baru.
2. Stop service yang connect ke database (lihat § Restore di atas).
3. Restore sesuai jenis database (PostgreSQL atau MariaDB, lihat section masing-masing
   di atas).
4. Verifikasi row count tabel utama (`tickets`, `ticket_articles`, `users`,
   `organizations`) cocok dengan ekspektasi sebelum start ulang service.
5. Start ulang service, verifikasi UI & search normal sebelum menganggap restore
   selesai.
