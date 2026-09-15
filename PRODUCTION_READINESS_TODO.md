# Kesiapan Produksi — Gap Analysis (belum dikerjakan)

⚠️ **Koreksi konteks (15 Sept 2026):** seluruh proyek ini (termasuk `zammad-audit` dan
`zammad-staging` di `Koi-Server-Dev`) adalah **sandbox riset/simulasi upgrade**, tidak
terhubung ke environment produksi nyata (yang berjalan di infrastruktur terpisah,
dengan data yang terus bertambah independen dari snapshot yang dipakai sandbox ini).
"Cutover produksi" di file ini berarti: **menerapkan playbook upgrade yang sudah
tervalidasi di sandbox ini ke environment produksi sungguhan nanti**, bukan
mengalihkan domain di dalam sandbox `Koi-Server-Dev` itu sendiri.

Hasil gap analysis terhadap seluruh dokumentasi (README, ROADMAP, DOWNTIME_ESTIMATE,
NOTES/CHANGELOG/RUNBOOK/TODO tiap hop) dibandingkan dengan best practice runbook,
incident management, dan kesiapan cutover produksi.

**Beda dengan "TODO — polish dokumentasi" di README.md:** file itu soal kerapian teks
yang SUDAH ADA (format, bahasa, duplikasi). File ini soal konten yang **belum ada sama
sekali**, dan sifatnya bukan kosmetik — ini tentang kesiapan operasional sebelum
playbook upgrade ini diterapkan ke environment produksi sungguhan, dengan data
pelanggan asli yang sudah bertambah sejak snapshot yang dipakai di sandbox ini.

**Kapan dikerjakan:** tidak harus menunggu hop 7.1.3 selesai seperti TODO polish —
sebagian item di sini (terutama kategori 3 soal keamanan data) sebaiknya mulai
dipertimbangkan lebih awal. Prioritaskan sebelum tanggal penerapan ke produksi nyata
ditetapkan, bukan sebelum hop terakhir sandbox selesai.

## Penilaian singkat

Untuk **latihan upgrade staging murni**, dokumentasi proyek ini sudah sangat memadai —
insiden dicatat detail dengan root cause, RUNBOOK hasil saringan bisa dieksekusi ulang,
estimasi downtime berbasis data nyata. Untuk **cutover produksi nyata dengan data
pelanggan sungguhan**, ada gap struktural yang nyata: belum ada gate keputusan formal
(go/no-go final, kriteria rollback), belum ada rencana pasca-cutover (monitoring,
komunikasi, dekomisioning), dan cakupan verifikasi masih berhenti di "UI terlihat
normal" padahal fungsi bisnis inti (email, otomasi) belum pernah diuji eksplisit di
satu pun hop.

---

## Top 5 prioritas (paling penting sebelum cutover produksi asli)

✅ **Item 1-6 di bawah sudah ditangani** — lihat [CUTOVER_CHECKLIST.md](CUTOVER_CHECKLIST.md)
(dibuat 15 Sept 2026). Disimpan di sini sebagai catatan asal-usul gap yang mendasari
checklist tersebut.

1. **Rencana monitoring pasca-cutover** — tanpa ini, regresi halus di produksi nyata
   (mis. email diam-diam gagal kirim) bisa tidak terdeteksi berhari-hari.
2. **Go/no-go checklist final untuk cutover produksi** — terpisah dari checklist
   verifikasi staging per-hop yang sudah ada.
3. **Kriteria KEPUTUSAN rollback** (bukan cuma langkah teknisnya) — pola nyata sejauh
   ini selalu "tambal di tempat lalu lanjut" (8 insiden hop 6.0→7.0 semua di-fix-forward,
   belum pernah rollback sungguhan); tanpa ambang eksplisit, berisiko debugging tanpa
   batas kalau terjadi di produksi asli.
4. **Verifikasi fungsi bisnis inti** — email kirim/terima, trigger/automation, minimal
   satu cek kolom yang sudah berganti nama (CHANGELOG hop 6.0→7.0 sendiri sudah
   memperingatkan ini tapi belum ada langkah verifikasi yang menutup peringatan itu).
5. **Rencana komunikasi ke stakeholder/user nyata** — `DOWNTIME_ESTIMATE.md` menghitung
   angka downtime detail, tapi tidak ada rencana memberi tahu agent/customer sungguhan
   soal maintenance window produksi asli.
6. **(Ditambahkan setelah koreksi konteks) Rencana pengambilan data produksi TERKINI**
   — seluruh pengujian di proyek ini pakai snapshot data produksi yang diambil di masa
   lalu, bukan data produksi yang berjalan saat ini. Sebelum playbook diterapkan nyata,
   perlu langkah eksplisit: ekspor ulang data produksi PALING BARU, verifikasi
   integritasnya (pelajaran dari insiden dump corrupt di `postgres-migration/NOTES.md`),
   baru jalankan playbook di atasnya — jangan asumsikan volume/karakteristik data akan
   sama persis dengan yang sudah diuji di sandbox (durasi reindex ES di
   `DOWNTIME_ESTIMATE.md` misalnya sangat bergantung jumlah tiket, yang kemungkinan
   sudah lebih banyak di produksi asli sekarang).

*(Kebijakan retensi dump backup berisi PII — kategori 3 di bawah — juga layak ditangani
segera meski tidak masuk 5 besar; ini lebih ke kepatuhan data daripada penghalang
teknis langsung.)*

---

## 1. Kelengkapan Runbook

- **Tidak ada kriteria DECISION untuk rollback**, hanya langkah "cara"-nya. Semua
  RUNBOOK punya bagian "Kalau perlu mundur" yang menjelaskan *bagaimana* (restore
  backup, hapus container), tapi tidak pernah menjelaskan *kapan* seharusnya benar-benar
  memutuskan mundur vs. lanjut fix-forward. → Tambahkan sub-bagian "Kapan harus mundur"
  dengan ambang konkret (mis. "kalau downtime keras >X jam", "kalau N percobaan fix
  gagal", "kalau migrasi merusak data").
- **Tidak ada owner/kontak dan jalur eskalasi** di RUNBOOK manapun. Wajar untuk usaha
  solo, tapi relevan untuk cutover nyata terutama proses berjam-jam (reindex ES) kalau
  sesuatu perlu keputusan saat eksekutor tidak tersedia. → Baris singkat
  "Owner: [nama/kontak]" di header tiap RUNBOOK.
- **Go/no-go sebelum mulai bersifat implisit** (checklist pre-flight), tidak menyatakan
  eksplisit konsekuensi kalau satu item gagal. → Untuk cutover produksi, buat gate
  tegas ("STOP kalau ini gagal") bukan cuma checklist centang.

## 2. Kesiapan Cutover Produksi

✅ **Sudah ditangani** — lihat [CUTOVER_CHECKLIST.md](CUTOVER_CHECKLIST.md) (dibuat 15
Sept 2026): gate go/no-go, kriteria keputusan rollback, verifikasi fungsional
diperluas, rencana monitoring pasca-cutover, dan komunikasi stakeholder semua sudah
disatukan di sana. Poin di bawah ini disimpan sebagai catatan asal-usul gap.

- **Tidak ada rencana monitoring pasca-cutover** — apa yang dipantau (error rate, log
  email channel, job scheduler, disk/ES health), berapa lama observasi, ambang yang
  memicu rollback pasca-live. → Dokumen singkat "Monitoring pasca-cutover".
- **Tidak ada go/no-go checklist final** yang menyatukan: playbook sudah tervalidasi
  ulang, rollback plan sudah jelas, stakeholder sudah diberi tahu, data produksi
  terkini (bukan snapshot sandbox) sudah disiapkan/diverifikasi integritasnya. →
  `CUTOVER_CHECKLIST.md` terpisah dari checklist per-hop di sandbox.
- **Tidak ada rencana komunikasi stakeholder/user nyata** soal maintenance window saat
  playbook ini benar-benar diterapkan ke produksi. → Section di `DOWNTIME_ESTIMATE.md`
  atau file terpisah: siapa perlu diberi tahu, kapan, lewat kanal apa.
- **Belum ada rencana konkret untuk mengambil & memverifikasi data produksi TERKINI**
  saat penerapan nyata nanti — data di sandbox ini cuma snapshot lama, produksi asli
  sudah bertambah. Perlu langkah eksplisit: ekspor data terbaru, verifikasi integritas,
  baru jalankan playbook di atasnya (bukan asumsi data sandbox = data final).

## 3. Penanganan Data & Keamanan Selama Migrasi

- **Tidak ada kebijakan retensi/pembersihan untuk file dump database** (`.sql.gz`)
  yang berisi data tiket/email/nama pelanggan asli — kapan dihapus, di mana boleh
  disimpan sementara, siapa boleh akses. (Beberapa dump nyata sempat jadi penyebab
  disk hampir penuh di hop-hop awal.)
- **Tidak ada catatan kontrol akses** ke environment staging yang sekarang melayani
  trafik pelanggan nyata (siapa punya SSH ke server, siapa bisa query database
  staging). "Catatan keamanan" di README cuma soal tidak menyimpan password di repo —
  topik berbeda dari kontrol akses server.

## 4. Dokumentasi Disaster Recovery / Backup

- **Tidak ada satu referensi backup/restore darurat** — langkah backup tersebar
  sebagai bullet pre-flight di 5 RUNBOOK berbeda. Kalau kondisi darurat produksi nyata
  butuh restore cepat, orang harus menelusuri RUNBOOK mana dulu. → `BACKUP_RESTORE.md`
  ringkas di root, cross-reference dari tiap RUNBOOK.
- **Pelajaran verifikasi integritas backup tidak digeneralisasi** — `postgres-migration`
  sudah menjadikannya langkah wajib eksplisit (`gzip -t`, insiden corrupt backup nyata),
  tapi 4 RUNBOOK hop lain pre-flight-nya cuma tulis "Backup database" tanpa syarat
  verifikasi apa pun. → Tambahkan satu baris verifikasi integritas ke semua RUNBOOK.

## 5. Kedalaman Testing/Verifikasi

Checklist "Verifikasi akhir" semua hop konsisten dangkal (login, search, versi benar).
Untuk helpdesk produksi nyata, ini timpang dibanding risiko sebenarnya — belum pernah
diuji eksplisit di hop manapun:
- **Channel email kirim/terima** — fungsi inti Zammad, kegagalan silent pasca-upgrade
  tidak akan terdeteksi checklist yang ada.
- **Trigger/automation terjadwal & Core Workflow** — beberapa hop menambah fitur ini,
  bisa diam-diam berhenti bekerja tanpa terdeteksi.
- **Kolom database yang berganti nama** — hop 6.0 CHANGELOG.md sudah memperingatkan
  soal ini tapi tidak ada langkah verifikasi konkret yang menutup peringatannya.
- **Webhook, permission/role, attachment file** — jadi fitur highlight beberapa hop
  tapi tidak pernah masuk checklist verifikasi.

→ Perluas "Verifikasi akhir" (khususnya hop terakhir 7.0→7.1.3 dan checklist cutover)
dengan pengujian fungsional konkret, bukan cuma visual UI.

## 6. Metadata Kepemilikan & Governance

Tidak ada pernyataan di mana pun soal siapa memelihara proyek ini, siapa punya akses
ke kredensial produksi, atau kontak untuk pertanyaan. Sesuai dugaan untuk usaha solo,
tapi karena repo publik, tetap dicatat — relevan kalau nanti perlu serah terima. →
Satu baris di README ("Dikelola oleh [nama], pertanyaan lewat [kanal]").

## 7. Kebersihan Repo Publik

- **Tidak ada pernyataan tujuan/audiens** untuk pengunjung acak GitHub — README
  langsung masuk ke detail teknis tanpa kalimat pembuka semacam "ini log teknis
  pribadi/internal, dibagikan sebagai referensi, bukan panduan upgrade Zammad
  generik". Relevan karena beberapa detail sangat spesifik ke server ini (disk
  timpang, resource dibagi banyak servis lain) yang bisa salah diterapkan mentah-mentah
  oleh pembaca lain.
- **Tidak ada file LICENSE** — dampak ringan karena isinya dokumentasi proses (bukan
  kode), tapi tetap dicatat sebagai absen.

## 8. Sintesis Lintas-Hop / Lessons Learned

ROADMAP.md § "Masalah yang berulang tiap hop" secara eksplisit hanya mencakup isu
build-from-source (gem yang di-yank, asset belum precompile, nama rake task berubah).
Ini sempit dibanding pola operasional yang benar-benar berulang lintas hop:
- **Index Elasticsearch stale/nyangkut memblokir rebuild** — terjadi berulang di hop
  4.0→5.0, 5.0→6.0, dan 6.0→7.0, masing-masing dicatat lokal per-hop tapi tidak pernah
  disatukan jadi satu entri "pola berulang".
- **Kejutan versi runtime dependency yang tidak terdeteksi dari riset dokumentasi
  resmi** — Redis hard dependency (hop 5.0→6.0) dan lonjakan requirement Redis ≥6
  (hop 6.0→7.0) adalah pola yang sama: requirement versi tidak lengkap di
  `.ruby-version`/dokumentasi resmi, baru ketahuan lewat kegagalan nyata saat boot.
- **"Container Up ≠ sehat"** — pelajaran eksplisit dari Insiden 7 hop 6.0→7.0
  (assets:precompile gagal diam-diam, semua halaman 500 meski container "Up"). Sangat
  relevan untuk hop 7.0→7.1.3 dan cutover produksi nyata, tapi cuma tertulis di NOTES
  satu hop, belum jadi prinsip umum.

→ Perluas ROADMAP.md § "Masalah yang berulang" (atau tambah section baru "Pelajaran
operasional lintas-hop") dengan 3 poin di atas.

## 9. Kelengkapan Gaya Diátaxis

- **Tidak ada "Tutorial"** — dokumen orientasi singkat untuk orang (termasuk diri
  sendiri di masa depan, atau pihak lain yang mengambil alih) yang belum pernah
  menyentuh proyek ini. README saat ini campuran reference+explanation, bukan
  panduan berpandu. → Tambahkan "Mulai di sini" pendek di README (5-10 menit
  orientasi: baca README → ROADMAP → tabel status → pilih hop).
- **CHANGELOG lintas-hop tersebar** di 5 file terpisah tanpa satu ringkasan/agregat
  "apa saja yang berubah total dari 3.4.0 ke 7.1.3". → Pertimbangkan satu tabel/daftar
  agregat perubahan skema besar lintas-hop.
