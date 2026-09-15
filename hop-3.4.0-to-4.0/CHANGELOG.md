# Hop 3.4.0 → 4.0 — Changelog

Format mengikuti [Keep a Changelog](https://keepachangelog.com/) (kategori
Added/Changed/Removed/Security), diadaptasi untuk konteks upgrade infrastruktur —
bukan rilis versi aplikasi sendiri.

Cakupan: rilis 3.5.0, 3.6.0, 4.0.0 (kita loncat langsung dari 3.4.0 ke 4.0.0, jadi
perubahan di 3.5.0/3.6.0 ikut termasuk). Sumber: `CHANGELOG.md` resmi tiap tag di
github.com/zammad/zammad, disaring — daftar lengkap (termasuk ratusan bug-fix kecil)
ada di link masing-masing rilis, tidak disalin semua ke sini.

## Added

- Tabel `mentions` — fitur **@mention** di tiket, agent dapat notifikasi
- Tabel `webhooks` — **Webhooks Admin UI**, kelola & lihat log aktivitas webhook
  langsung dari admin panel (sebelumnya cuma bisa lewat API)
- Tabel `data_privacy_tasks` — fitur Data Privacy/GDPR
- **Integrasi GitHub & GitLab** — link tiket ke issue GitHub/GitLab
- **Generic SSO button** di halaman login
- **Import archive mailbox** — bisa import email dari file arsip mailbox
- **Exchange Online MFA** untuk IMAP & SMTP (penting untuk channel email yang pakai MFA)
- **Dukungan responsive Tablet/Mobile** untuk UI Zammad
- Upload/download S/MIME certificate chain dari satu file
- Kolom `permissions.allow_signup`

## Changed

- Ruby 2.6.5 → **2.6.6**, Rails → **5.2.4.5** (Bundler ~1.17, base OS Debian Stretch,
  ES 6.8.23, database MariaDB — tidak berubah)
- `datetime_precision`: presisi kolom datetime dinaikkan (limit 3) di ~15 tabel
  (`taskbars`, `delayed_jobs`, `sessions`, `knowledge_base_*`, `oauth_*`, dll)
- `stats_stores`: kolom `o_id`, `stats_store_object_id`,
  `related_stats_store_object_id` dihapus, diganti asosiasi polymorphic
  (`stats_storable_type`/`stats_storable_id`)
- **Reauthentication Google/Microsoft 365** tanpa perlu setup ulang channel dari nol
- Perbaikan reliabilitas `rake searchindex:rebuild` saat Elasticsearch belum
  terkonfigurasi
- Password security default dinaikkan untuk instalasi baru
- Setting-setting baru (insert row, bukan perubahan struktur): import archive,
  karakter khusus, format nama pengirim, max size ES, dll.

## Removed

- `data_privacy_tasks.name` (dihapus lagi setelah sempat ditambahkan di migrasi yang
  sama rentang ini)
- `templates.user_id` dan `text_modules.user_id` (foreign key)

**Detail lengkap (termasuk bug-fix minor):**
[3.5.0](https://github.com/zammad/zammad/blob/3.5.0/CHANGELOG.md) ·
[3.6.0](https://github.com/zammad/zammad/blob/3.6.0/CHANGELOG.md) ·
[4.0.0](https://github.com/zammad/zammad/blob/4.0.0/CHANGELOG.md)
