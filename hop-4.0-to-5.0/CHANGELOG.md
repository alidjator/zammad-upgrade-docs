# Hop 4.0 → 5.0 — Ringkasan Perubahan

Cakupan: rilis 4.1.0 dan 5.0.0 (kita loncat langsung dari 4.0.0 ke 5.0.0, jadi
perubahan di 4.1.0 ikut termasuk). Sumber: `CHANGELOG.md` resmi tiap tag di
github.com/zammad/zammad, disaring.

## 1. Environment (before → after)

| Komponen | 4.0.0 | 5.0.0 |
|---|---|---|
| Ruby | 2.6.6 | **2.7.4** |
| Rails | 5.2.4.5 | **6.0.4.1** |
| Bundler | ~1.17 | **2.2.20** |
| Base OS image | Debian Stretch | **Debian Buster** (tidak ada tag ruby:2.7.4-stretch) |
| Elasticsearch | 6.8.23 | **7.17.28** (wajib naik ≥7.8) |
| Database (staging) | MariaDB 11.8.3 (host) | **MariaDB 10.11 (container terpisah)** — lihat
  keputusan besar di NOTES.md, karena bug versi-gap Rails 6.0 vs MariaDB 11.x |

## 2. Schema — perubahan tabel/kolom (dari ~60 migrasi)

**Tabel baru:**
- `core_workflows` — fitur besar **Core Workflow** (lihat bagian fitur di bawah)

**Perubahan kolom pada tabel existing:**
- `chats.whitelisted_websites` → di-rename jadi `chats.allowed_websites` (inclusive
  wording cleanup)
- `data_privacy_tasks` diubah strukturnya (`change_table`, preferences jadi text)
- `knowledge_bases.color_header_link` ditambahkan (sempat dengan default warna,
  default-nya dihapus lagi di migrasi berikutnya karena ada typo bug yang diperbaiki)
- `active_storage_attachments` — foreign key ke `active_storage_blobs` ditambahkan
- Setting-setting baru terkait: session timeout (multiple migrations: init, scheduler,
  update defaults, dropdown, description), Core Workflow permission, Microsoft 365
  tenants, Jira config, CheckMK wording

## 3. Fitur — highlight (dari CHANGELOG resmi 4.1.0/5.0.0)

- **Core Workflow** — fitur BESAR baru: admin bisa bikin aturan kondisional untuk
  form tiket (tampilkan/sembunyikan/wajibkan field berdasarkan kondisi lain), tanpa
  perlu custom code. Ini salah satu fitur paling signifikan di rilis 5.0.
- **Blokir script content (JavaScript) di email** — perbaikan keamanan penting,
  mencegah XSS lewat isi email yang dirender di UI
- **Read-only custom objects** — bisa buat custom field yang read-only
- **MessageBird integration** — channel komunikasi baru
- **KB Answer tagging** & seleksi berdasarkan kode negara (`DE`, `ES`, dst.)
- **Session timeout diperbaiki total**: default 4 minggu, verifikasi cuma sekali/jam
  (bukan tiap request — lebih ringan), ditampilkan dalam menit bukan detik
- **Granular permission untuk channel Google** (admin bisa kasih akses spesifik,
  tidak semua-atau-tidak-sama-sekali)
- **Deteksi follow-up Jira** otomatis
- Kirim/forward email internal (4.1.0)
- Visualisasi & unlock user yang ter-lock langsung dari UI
- Bulk option di extended search
- Perbaikan `rake zammad:package:migrate` untuk linked package (relevan untuk
  environment custom/self-hosted seperti kita)

**Detail lengkap (termasuk bug-fix minor):**
[4.1.0](https://github.com/zammad/zammad/blob/4.1.0/CHANGELOG.md) ·
[5.0.0](https://github.com/zammad/zammad/blob/5.0.0/CHANGELOG.md)

## Catatan untuk admin/user Zammad (bukan cuma developer)

Setelah upgrade ini, ada baiknya sosialisasikan ke tim agent/admin:
- **Core Workflow** tersedia — kalau tim punya kebutuhan form kondisional yang selama
  ini "susah diatur", ini waktunya explore fitur ini di Admin → Manage → Core Workflows.
- Default **session timeout berubah jadi 4 minggu** — kalau kebijakan keamanan internal
  butuh timeout lebih pendek, perlu diset manual di Admin → Security.
- Kalau ada yang pakai channel **Google**, cek ulang permission granular yang baru —
  role lama mungkin perlu di-assign ulang permission spesifik channel Google.
