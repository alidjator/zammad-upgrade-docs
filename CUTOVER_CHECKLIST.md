# Checklist Cutover Produksi Nyata

**Konteks penting:** dokumen ini untuk saat playbook upgrade yang sudah divalidasi di
sandbox riset (`Koi-Server-Dev`, lihat README.md) **diterapkan ke environment produksi
sungguhan** — infrastruktur terpisah, dengan data yang terus bertambah sejak snapshot
yang dipakai untuk pengujian di sandbox. Ini BUKAN checklist untuk mengalihkan domain
di dalam sandbox itu sendiri.

Checklist ini melengkapi (bukan menggantikan) `RUNBOOK.md` per hop — RUNBOOK berisi
langkah teknis tiap hop yang sudah terbukti berhasil di sandbox, checklist ini
menyatukan semuanya jadi satu alur keputusan untuk eksekusi produksi nyata.

---

## 0. Sebelum menjadwalkan tanggal cutover

- [ ] **Ekspor data produksi TERKINI** (bukan snapshot lama yang dipakai sandbox) dan
      **verifikasi integritasnya** (`gzip -t` atau setara — lihat insiden dump corrupt
      di [postgres-migration/NOTES.md](postgres-migration/NOTES.md), jangan asumsikan
      sukses cuma karena tidak ada error terlihat)
- [ ] Catat ukuran data terkini (jumlah tiket, artikel, user) — **bandingkan dengan
      angka sandbox** (161.894 tiket / 1.031.898 artikel saat snapshot diambil).
      Kalau jauh lebih besar, **skalakan ulang estimasi waktu** di
      [DOWNTIME_ESTIMATE.md](DOWNTIME_ESTIMATE.md) (reindex ES di sandbox terbukti
      berbanding lurus dengan jumlah tiket — jangan asumsikan durasi sama)
- [ ] Konfirmasi spesifikasi server produksi nyata (disk, RAM, CPU) — proyek sandbox
      ini mengalami krisis disk di **hampir setiap hop** karena resource server
      terbatas/dipakai bersama; pastikan produksi nyata punya headroom disk memadai
      (idealnya >20GB bebas sebelum mulai, dan disk tidak mendekati 90% selama proses)
- [ ] Rollback data produksi nyata SAAT INI (sebelum upgrade apa pun dimulai) — backup
      penuh, disimpan di luar server, integritas terverifikasi
- [ ] Owner/eksekutor proses ditetapkan jelas, dengan kontak eskalasi kalau proses
      berjam-jam (reindex ES) butuh keputusan saat eksekutor utama tidak tersedia
- [ ] Stakeholder (agent/tim support, kalau relevan pelanggan) sudah diberi tahu
      jadwal maintenance window — lihat § 5 Komunikasi Stakeholder di bawah
- [ ] Playbook (RUNBOOK per hop) sudah dibaca ulang dari awal, termasuk semua catatan
      insiden di NOTES.md tiap hop — jangan asumsikan hafal dari eksekusi sandbox

**Gate keras:** kalau salah satu item di atas gagal/belum siap, **JANGAN mulai
cutover** — jadwalkan ulang setelah terpenuhi. Ini bukan checklist "usahakan", tapi
syarat mutlak sebelum ada tindakan yang menyentuh sistem produksi nyata.

## 1. Urutan eksekusi

Ikuti urutan hop yang sama seperti yang divalidasi di sandbox — **tidak boleh
melompati major version**:

```
3.4.0 → 4.0 → 5.0 → 6.0 → [migrasi MariaDB→PostgreSQL] → 7.0 → 7.1.3
```

Untuk tiap hop, ikuti `hop-X-to-Y/RUNBOOK.md` yang relevan. Catatan penting hasil
sandbox yang WAJIB diperhatikan (ringkasan — detail lengkap di NOTES.md tiap hop):

- Base image Debian era-appropriate per hop (lihat `ROADMAP.md`)
- `pkg-config` + `libimlib2-dev` + `ENV CI=true` wajib di Dockerfile sejak hop 6.0→7.0
- Redis ≥6 wajib sejak hop 6.0→7.0 (gunakan `redis:7-alpine` atau lebih baru)
- **Image `zammad-app`/`websocket`/`scheduler` WAJIB pakai `image:` yang SAMA** di
  `docker-compose.yml` — kalau tidak, build jadi 3x lebih lama (terbukti terjadi di
  SEMUA hop sandbox sampai baru disadari & diperbaiki di hop 6.0→7.0)
- `assets:precompile` dijalankan di runtime (`command:`), bukan build time
- **Verifikasi `public/assets/` benar-benar berisi `application-*.css` setelah
  container pertama kali `Up`** — container bisa "Up" padahal asset pipeline gagal
  diam-diam (Insiden 7, hop 6.0→7.0)
- Cek `_cat/indices` bersih sebelum `searchindex:rebuild` — index stale dari
  percobaan gagal sebelumnya bisa memblokir rebuild
- Pantau disk selama reindex — ES bisa masuk mode `read_only_allow_delete` kalau
  disk lewat flood-stage watermark, block ini TIDAK otomatis lepas
- Migrasi database Zammad historis kadang punya **bug urutan** (migrasi lama butuh
  tabel yang baru dibuat migrasi jauh lebih baru — lihat Insiden 6 hop 6.0→7.0). Kalau
  `db:migrate` gagal dengan `PG::UndefinedTable` pada migrasi yang seharusnya tidak
  berhubungan, cek pola ini sebelum panik.

## 2. Kriteria KEPUTUSAN rollback (bukan cuma langkah teknisnya)

Pola nyata di sandbox: **semua 8+ insiden selalu ditangani dengan fix-forward**
(tambal di tempat, lanjut), belum pernah ada rollback sungguhan. Itu valid di sandbox
(risiko rendah, bisa dicoba berkali-kali), tapi **tidak boleh jadi default di produksi
nyata** tanpa batas waktu. Tetapkan ambang ini SEBELUM mulai, bukan diputuskan di
tengah tekanan:

- [ ] **Batas waktu**: kalau downtime keras (build+migrate, biasanya di bawah 1 jam
      berdasarkan sandbox — tapi verifikasi ulang untuk data produksi nyata) melebihi
      ambang yang disepakati (mis. 2x estimasi), STOP dan evaluasi — jangan terus
      coba tanpa batas.
- [ ] **Batas percobaan**: kalau satu masalah yang sama gagal diperbaiki setelah N
      percobaan (sepakati N di awal, mis. 3), STOP dan rollback — jangan lanjut
      trial-and-error di produksi nyata.
- [ ] **Integritas data**: kalau ada indikasi migrasi merusak/menghilangkan data
      (row count tidak cocok, error data corruption), **rollback WAJIB**, bukan opsi —
      tidak ada fix-forward yang aman untuk kerusakan data.
- [ ] Cara rollback: restore dari backup pre-cutover (§0), revert DNS/reverse-proxy ke
      sistem produksi lama. Pastikan proses restore ini SENDIRI sudah pernah diuji
      (bukan cuma diasumsikan bekerja) sebelum hari-H.

## 3. Verifikasi fungsional (lebih dalam dari checklist staging per-hop)

Checklist "Verifikasi akhir" di tiap RUNBOOK sandbox cukup dangkal (login, search,
versi tampil). Untuk produksi nyata, perluas dengan pengujian fungsional konkret:

- [ ] UI menampilkan versi target, login normal, buka/edit tiket normal
- [ ] Search tiket mengembalikan hasil akurat (query langsung ke ES untuk sampel
      tiket spesifik, bandingkan `_count` dengan jumlah tiket sumber — pola yang
      terbukti di hop 6.0→7.0)
- [ ] **Kirim & terima email test lewat channel email produksi** — fungsi inti
      Zammad, kegagalan silent pasca-upgrade tidak akan terdeteksi checklist visual
- [ ] **Satu trigger/automation terjadwal terbukti jalan** (bukan cuma ada di daftar)
- [ ] **Satu webhook (kalau dipakai) terbukti terkirim**
- [ ] **Satu permission/role non-admin diverifikasi** — pastikan behavior akses tidak
      berubah tak terduga
- [ ] Upload & download satu attachment file
- [ ] WebSocket real-time update jalan (cek `/cable` di browser devtools)
- [ ] Kolom database yang diketahui berganti nama antar hop (lihat CHANGELOG tiap
      hop) — cek satu integrasi/laporan yang bergantung pada kolom tersebut, kalau ada

## 4. Monitoring pasca-cutover

- [ ] **Durasi observasi ketat**: minimal 24-72 jam pertama setelah cutover, jangan
      anggap selesai begitu UI terlihat normal
- [ ] **Yang dipantau**: error rate aplikasi (log Rails), log pengiriman/penerimaan
      email channel, status job scheduler (`background-worker`), disk & health
      Elasticsearch, response time UI
- [ ] **Ambang yang memicu evaluasi rollback**: sepakati angka konkret sebelum hari-H
      (mis. error rate >X% dalam 1 jam, email gagal kirim >Y kali berturut-turut, disk
      >90% tanpa penjelasan)
- [ ] Setelah observasi awal aman, lanjutkan pemantauan rutin (bukan intensif) minimal
      1 minggu sebelum dianggap benar-benar stabil

## 5. Komunikasi stakeholder

- [ ] **Siapa** yang perlu diberi tahu: agent/tim support internal (wajib), pelanggan
      (kalau maintenance window terlihat/mengganggu akses mereka)
- [ ] **Kapan**: idealnya H-3 sampai H-1 sebelum window, plus pengingat di hari-H
- [ ] **Lewat kanal apa**: (isi sesuai kebiasaan organisasi — email internal, pesan
      broadcast, banner di sistem lama, dll.)
- [ ] **Isi pesan minimal**: tanggal & jam window, estimasi durasi (build+migrate
      "keras" vs. reindex "lunak" — lihat DOWNTIME_ESTIMATE.md), dampak yang
      diharapkan (search mungkin tidak akurat sementara selama reindex, tiket tetap
      bisa dibuka/diedit begitu migrasi selesai)
- [ ] Kabar penutup setelah cutover dikonfirmasi stabil (bukan cuma kabar saat mulai)

## 6. Setelah stabil — dekomisioning

- [ ] Tetapkan durasi "masa observasi" sebelum sistem lama (versi sebelum upgrade)
      benar-benar dimatikan/dihapus — jangan buru-buru, ini jaring pengaman rollback
      termurah
- [ ] Setelah masa observasi terlewati tanpa insiden, rencanakan dekomisioning sistem
      lama: retensi data (berapa lama disimpan sebagai arsip), kapan dihapus permanen
- [ ] Update dokumentasi ini (dan README.md) untuk mencerminkan status akhir setelah
      cutover nyata benar-benar selesai
