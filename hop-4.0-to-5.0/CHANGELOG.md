# Hop 4.0 → 5.0 — Changelog

Format mengikuti [Keep a Changelog](https://keepachangelog.com/) (kategori
Added/Changed/Security dipakai di hop ini), diadaptasi untuk konteks upgrade
infrastruktur — bukan rilis versi aplikasi sendiri.

Cakupan: rilis 4.1.0 dan 5.0.0 (kita loncat langsung dari 4.0.0 ke 5.0.0, jadi
perubahan di 4.1.0 ikut termasuk). Sumber: `CHANGELOG.md` resmi tiap tag di
github.com/zammad/zammad, disaring.

## Added

- Tabel `core_workflows` — fitur BESAR baru **Core Workflow**: admin bisa bikin
  aturan kondisional untuk form tiket (tampilkan/sembunyikan/wajibkan field
  berdasarkan kondisi lain), tanpa perlu custom code. Salah satu fitur paling
  signifikan di rilis 5.0.
- **Read-only custom objects** — bisa buat custom field yang read-only
- **MessageBird integration** — channel komunikasi baru
- **KB Answer tagging** & seleksi berdasarkan kode negara (`DE`, `ES`, dst.)
- **Granular permission untuk channel Google** (admin bisa kasih akses spesifik,
  tidak semua-atau-tidak-sama-sekali)
- **Deteksi follow-up Jira** otomatis
- Kirim/forward email internal (4.1.0)
- Visualisasi & unlock user yang ter-lock langsung dari UI
- Bulk option di extended search
- `active_storage_attachments` — foreign key ke `active_storage_blobs`
- `knowledge_bases.color_header_link`

## Changed

- Ruby 2.6.6 → **2.7.4**, Rails 5.2.4.5 → **6.0.4.1**, Bundler ~1.17 → **2.2.20**,
  base OS Debian Stretch → **Debian Buster** (tidak ada tag `ruby:2.7.4-stretch`),
  Elasticsearch 6.8.23 → **7.17.28** (wajib naik ≥7.8)
- **Database staging pindah dari MariaDB 11.8.3 (host) ke MariaDB 10.11 (container
  terpisah)** — lihat keputusan besar di [NOTES.md](NOTES.md), karena bug versi-gap
  Rails 6.0 vs MariaDB 11.x
- `chats.whitelisted_websites` → di-rename jadi `chats.allowed_websites` (inclusive
  wording cleanup)
- `data_privacy_tasks` diubah strukturnya (`change_table`, preferences jadi text)
- **Session timeout diperbaiki total**: default 4 minggu, verifikasi cuma sekali/jam
  (bukan tiap request — lebih ringan), ditampilkan dalam menit bukan detik
- Perbaikan `rake zammad:package:migrate` untuk linked package (relevan untuk
  environment custom/self-hosted seperti kita)
- Setting-setting baru terkait: session timeout (multiple migrations: init,
  scheduler, update defaults, dropdown, description), Core Workflow permission,
  Microsoft 365 tenants, Jira config, CheckMK wording

## Security

- **Blokir script content (JavaScript) di email** — perbaikan keamanan penting,
  mencegah XSS lewat isi email yang dirender di UI

**Detail lengkap (termasuk bug-fix minor):**
[4.1.0](https://github.com/zammad/zammad/blob/4.1.0/CHANGELOG.md) ·
[5.0.0](https://github.com/zammad/zammad/blob/5.0.0/CHANGELOG.md)

## Catatan untuk admin/user Zammad (bukan cuma developer)

Setelah upgrade ini, ada baiknya sosialisasikan ke tim agent/admin:
- **Core Workflow** tersedia — jika tim punya kebutuhan form kondisional yang selama
  ini "susah diatur", ini waktunya explore fitur ini di Admin → Manage → Core Workflows.
- Default **session timeout berubah jadi 4 minggu** — jika kebijakan keamanan internal
  butuh timeout lebih pendek, perlu diset manual di Admin → Security.
- Jika ada yang pakai channel **Google**, cek ulang permission granular yang baru —
  role lama mungkin perlu di-assign ulang permission spesifik channel Google.
