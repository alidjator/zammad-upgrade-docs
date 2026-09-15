# Hop 5.0 → 6.0 — Changelog

Format mengikuti [Keep a Changelog](https://keepachangelog.com/) (kategori
Added/Changed/Removed dipakai di hop ini), diadaptasi untuk konteks upgrade
infrastruktur — bukan rilis versi aplikasi sendiri.

Cakupan: rilis 5.1.0 sampai 5.4.1, dan 6.0.0 (kita loncat langsung dari 5.0.0 ke
6.0.0, jadi seluruh rilis minor 5.x ikut termasuk). Karena rentangnya panjang (5
rilis minor), highlight fitur di bawah cuma diambil dari entri 6.0.0 sendiri — untuk
detail rilis 5.1-5.4 lihat link compare di bagian bawah.

## Added

- Tabel `ldap_sources` — dukungan **multi-LDAP** (sebelumnya cuma 1 sumber LDAP)
- Tabel `ticket_shared_draft_zooms`, `ticket_shared_draft_starts` — fitur **shared
  draft** tiket
- Tabel `knowledge_base_permissions` — permission granular per KB
- Tabel `public_links`, `user_overview_sortings`
- **Default Agent Notification Settings** — admin bisa set default notifikasi untuk
  semua agent baru
- **Deteksi duplikat tiket** — bantu agent/customer supaya tidak submit tiket yang
  sama dua kali
- **Trigger berbasis waktu** diperluas (time-based events baru)
- **Pre-defined webhooks** — template webhook siap pakai (tidak perlu setup manual
  dari nol)
- **Custom payload untuk webhooks** + **HTTP BasicAuth untuk webhooks**
- **Session conditions** bisa ditambahkan ke semua object (bukan cuma tiket)

## Changed

- Ruby 2.7.4 → **3.1.3**, Rails 6.0.4.1 → **6.1.7.3**, Bundler 2.2.20 → **2.4.1**
- **Node.js jadi wajib versi 18.x LTS via NodeSource** (sebelumnya 10.24 bawaan
  Debian Buster, tidak dipakai serius) — `package.json` mensyaratkan `>=16`
- **Build tool JS: Sprockets saja → Sprockets + Vite** (baru)
- `tokens.name` → `token`, `tokens.label` → `name` (rename membingungkan, perhatikan
  jika ada integrasi yang query langsung ke kolom ini)
- `email_addresses.realname` → `name`
- Banyak `change_column` presisi/panjang string (translations, organizations,
  http_logs)
- Perbaikan UX tablet (sidebar tidak collapse otomatis)
- (Base OS Debian Buster, Elasticsearch 7.17.28, database MariaDB 10.11 — tidak
  berubah di hop ini)

## Removed

- Tabel `notifications` (`DropNotificationsTable`)
- Tabel `karma_activity_logs`, `karma_activities`, `karma_users` (`DropKarma` —
  **fitur karma/gamifikasi dihapus total**)

**Detail lengkap (termasuk rilis 5.1.0-5.4.1 yang kita lewati):**
[Compare 5.0.0...6.0.0](https://github.com/zammad/zammad/compare/5.0.0...6.0.0) ·
[6.0.0 CHANGELOG](https://github.com/zammad/zammad/blob/6.0.0/CHANGELOG.md)

## Catatan untuk admin/user Zammad

- **Fitur Karma/gamifikasi sudah dihapus total** dari Zammad — jika tim pernah pakai
  ini, datanya sudah hilang bersama tabelnya (tidak ada cara mundur selain restore
  backup lama)
- Jika ada integrasi eksternal yang baca langsung tabel `tokens` atau
  `email_addresses` di database (bukan lewat API resmi), **cek ulang** — nama
  kolomnya berubah di hop ini
- **Pre-defined webhooks** baru tersedia — jika tim sering setup webhook manual
  berulang, worth dicek apakah sudah ada template siap pakai untuk kasusnya
