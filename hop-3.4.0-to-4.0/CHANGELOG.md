# Hop 3.4.0 → 4.0 — Ringkasan Perubahan

Cakupan: rilis 3.5.0, 3.6.0, 4.0.0 (kita loncat langsung dari 3.4.0 ke 4.0.0, jadi
perubahan di 3.5.0/3.6.0 ikut termasuk). Sumber: `CHANGELOG.md` resmi tiap tag di
github.com/zammad/zammad, disaring — daftar lengkap (termasuk ratusan bug-fix kecil)
ada di link masing-masing rilis, tidak disalin semua ke sini.

## 1. Environment (before → after)

| Komponen | 3.4.0 | 4.0.0 |
|---|---|---|
| Ruby | 2.6.5 | 2.6.6 |
| Rails | 5.2.x | 5.2.4.5 |
| Bundler | ~1.17 | ~1.17 (tidak berubah) |
| Base OS image | Debian Stretch | Debian Stretch (tidak berubah) |
| Elasticsearch | 6.8.23 (tidak berubah di hop ini) | 6.8.23 |
| Database | MariaDB (bebas versi) | MariaDB (bebas versi, belum ada batasan ketat) |

## 2. Schema — perubahan tabel/kolom (dari 33 migrasi)

**Tabel baru:**
- `data_privacy_tasks` — fitur Data Privacy/GDPR (lalu kolom `name` dihapus lagi di
  migrasi berikutnya)
- `mentions` — fitur @mention di tiket
- `webhooks` — Webhooks Admin View

**Perubahan kolom pada tabel existing (bukan sekadar tambah tabel):**
- `datetime_precision`: presisi kolom datetime dinaikkan (limit 3) di ~15 tabel
  (`taskbars`, `delayed_jobs`, `sessions`, `knowledge_base_*`, `oauth_*`, dll)
- `stats_stores`: kolom `o_id`, `stats_store_object_id`, `related_stats_store_object_id`
  dihapus, diganti asosiasi polymorphic (`stats_storable_type`/`stats_storable_id`)
- `permissions.allow_signup` ditambahkan
- `templates.user_id` dan `text_modules.user_id` (foreign key) dihapus

**Setting-setting baru** (insert row, bukan perubahan struktur): import archive, karakter
khusus, format nama pengirim, max size ES, dll.

## 3. Fitur — highlight (dari CHANGELOG resmi 3.5.0/3.6.0/4.0.0)

- **Mentions & Subscriptions** — agent bisa di-@mention di tiket, dapat notifikasi
- **Webhooks Admin UI** — kelola & lihat log aktivitas webhook langsung dari admin panel
  (sebelumnya cuma bisa lewat API)
- **Integrasi GitHub & GitLab** — link tiket ke issue GitHub/GitLab
- **Generic SSO button** di halaman login
- **Import archive mailbox** — bisa import email dari file arsip mailbox
- **Exchange Online MFA** untuk IMAP & SMTP (penting untuk channel email yang pakai MFA)
- **Dukungan responsive Tablet/Mobile** untuk UI Zammad
- **Reauthentication Google/Microsoft 365** tanpa perlu setup ulang channel dari nol
- Perbaikan reliabilitas `rake searchindex:rebuild` saat Elasticsearch belum terkonfigurasi
- Password security default dinaikkan untuk instalasi baru
- Upload/download S/MIME certificate chain dari satu file

**Detail lengkap (termasuk bug-fix minor):**
[3.5.0](https://github.com/zammad/zammad/blob/3.5.0/CHANGELOG.md) ·
[3.6.0](https://github.com/zammad/zammad/blob/3.6.0/CHANGELOG.md) ·
[4.0.0](https://github.com/zammad/zammad/blob/4.0.0/CHANGELOG.md)
