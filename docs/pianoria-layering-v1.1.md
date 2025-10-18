# PianoRIA — Layering & Spesifikasi Aplikasi (v1.1)

> Zona waktu seluruh sistem: **Asia/Jakarta (WIB)**  ·  Mata uang: **IDR**  ·  Target utama: Android (iOS menyusul)

Dokumen ini menjadi pegangan tim produk, desain, dan engineering dalam membangun PianoRIA versi 1.1. Konten difokuskan pada struktur layer, kebutuhan fungsional per peran, alur kunci end-to-end, aturan harga/biaya, serta edge case prioritas. Semua ketentuan di sini diasumsikan sudah melewati validasi bisnis dan siap diterjemahkan ke backlog buildable.

---

## 1. Ringkasan Konsep Produk

PianoRIA adalah aplikasi mobile yang mempertemukan guru piano dengan siswa dan orang tua, lengkap dengan akses belajar, penjadwalan, verifikasi kehadiran, serta marketplace buku piano dengan checkout di dalam aplikasi. Akses booking dibuka melalui **RIA Access (Student Pass)** berlangganan IDR 59.900/30 hari. Guru menentukan harga sesi, lokasi pengajaran (datang ke rumah, belajar di studio guru, atau keduanya), jadwal, kebijakan pembatalan, dan dapat merekomendasikan buku dari marketplace. Pembayaran utama menggunakan transfer/virtual account (VA). Kehadiran les diverifikasi menggunakan GPS dan foto bertimestamp. Orang tua memantau jadwal serta progress belajar anak setiap sesi.

---

## 2. Layering Arsitektur

| Layer | Tanggung jawab utama | Komponen contoh |
| --- | --- | --- |
| **Presentation (Mobile/Web)** | UI multi-peran (Student, Parent, Teacher, Seller) dengan navigasi berbasis tab; interaksi chat; formulir booking/progress; checkout marketplace. | Aplikasi React Native / Flutter (mobile), Admin SPA (web). |
| **Application Services** | Mengorkestrasi use case: pembelian pass, booking les, verifikasi kehadiran, penarikan saldo, checkout buku, sengketa. Menjaga aturan status & RBAC. | Modul service per domain (`BookingService`, `MarketplaceService`, `PaymentService`, `ProgressService`). |
| **Domain** | Model inti & aturan bisnis: StudentPass, Booking, Attendance, ProgressLog, BookOrder, Komisi Marketplace. Mengenkapsulasi validasi (misal radius GPS, fee tier). | Entitas/domain logic; aggregate root (Booking, BookOrder). |
| **Infrastructure** | Integrasi eksternal: OTP, penyimpanan foto (object storage), layanan pembayaran (VA/transfer), push notification, peta/GPS, pencatatan audit. | Adapter OTP provider, Payment Gateway connector, Storage SDK, Notification queue, Postgres/Prisma ORM. |

> **Catatan**: Admin console tetap web-based, memanfaatkan komponen Presentation + Application Service yang sama lewat API internal.

---

## 3. Peran Pengguna & Kapabilitas

### 3.1 Student (Siswa)
- Beli & kelola **RIA Access** (Student Pass).
- Jelajahi daftar guru, gunakan filter, lihat profil detail.
- Ajukan booking, unggah bukti pembayaran, bayar sesi, beri rating & ulasan.
- Terima rekomendasi buku, tambahkan ke keranjang marketplace, checkout, unggah bukti transfer.
- Dapat dihubungkan dengan akun orang tua.

### 3.2 Parent/Guardian (Orang Tua)
- Melihat status pass anak, jadwal sesi, riwayat booking.
- Mengakses Progress log setiap sesi; komentar/tanya balik ke guru.
- Opsional: melakukan pembayaran pass/sesi untuk anak.
- Checkout marketplace buku untuk anak.

### 3.3 Teacher (Guru)
- Lengkapi profil publik (bio, pengalaman, kurikulum, preferensi lokasi, harga, badge verifikasi).
- Atur slot ketersediaan, kebijakan pembatalan, tarif transport.
- Kelola permintaan booking (terima, reschedule, tolak) & pantau jadwal harian.
- Mulai & akhiri sesi (verifikasi GPS + foto), isi Progress log, kirim rekomendasi buku.
- Pantau saldo penghasilan, riwayat komisi & penarikan.

### 3.4 Seller (Penjual Buku)
- CRUD listing buku (stok, varian, ongkir, gambar).
- Kelola pesanan (status, input resi, konfirmasi pengiriman).
- Lihat biaya platform & payout yang dijadwalkan.

### 3.5 Admin (Backoffice Web)
- Moderasi & KYC (guru, seller), kelola konfigurasi biaya, radius GPS, harga pass.
- Audit booking, pembayaran, pesanan buku, progres.
- Menangani sengketa & refund.
- Mengelola konten (review, listing) & analytics.

RBAC wajib: satu akun boleh memiliki multi-peran, hak akses mengikuti role yang sedang diaktifkan.

---

## 4. Alur Navigasi Inti per Peran

### Student & Parent (mobile)
1. **Beranda** — status pass, pengingat jadwal, quick actions (Beli Pass, Cari Guru, Progress).
2. **Cari Guru** — daftar & filter, akses profil detail, tombol chat (butuh pass aktif).
3. **Pemesanan** — tab "Baru", "Terkonfirmasi", "Riwayat" dengan status booking.
4. **Progress** — daftar log per sesi (Parent read-only, Student bisa komentar bila diizinkan guru).
5. **Marketplace Buku** — katalog, rekomendasi dari guru, keranjang, checkout.
6. **Pembayaran / Wallet** — daftar pembayaran pass/sesi/buku, status verifikasi, unggah bukti.
7. **Profil & Pengaturan** — data diri, pengaturan notifikasi, tautan Parent.

### Teacher (mobile)
1. **Beranda** — ringkasan sesi hari ini, tindakan (mulai sesi, isi progress tertunda).
2. **Murid Saya** — daftar relasi aktif, cepat akses chat & progress.
3. **Jadwal** — kelola slot ketersediaan mingguan, blok tanggal khusus.
4. **Pemesanan** — permintaan baru, status berjalan.
5. **Progress** — form per booking untuk diisi ≤24 jam.
6. **Rekomendasi Buku** — pilih listing marketplace, kirim ke murid.
7. **Penghasilan** — saldo, komisi platform, riwayat penarikan.
8. **Profil & Pengaturan** — edit profil publik, preferensi, info payout.

### Seller (mobile/web adaptif)
1. **Beranda** — statistik order & fee terbaru.
2. **Produk Saya** — kelola listing, stok, varian, ongkir.
3. **Pesanan** — status order (Dibayar → Diproses → Dikirim → Diterima → Selesai), aksi input resi.
4. **Pembayaran** — riwayat payout, biaya platform yang dipotong.
5. **Profil & Pengaturan** — info toko, KYC, rekening payout.

### Admin (web backoffice)
- **Dashboard**, **Users & KYC**, **Teachers**, **Bookings**, **Marketplace & Pesanan**, **Pembayaran & Payout**, **Sengketa**, **Moderasi Konten**, **Analytics**, **Konfigurasi**.

---

## 5. Modul & Fitur Utama

### 5.1 Student Pass (RIA Access)
- Harga default IDR 59.900 untuk 30 hari (durasi bisa dikonfigurasi admin).
- Status: `Active`, `Expired`, `Pending Payment`.
- Perpanjangan otomatis kecuali dibatalkan manual sebelum tanggal perpanjangan berikutnya.
- Pembayaran via transfer/VA; dukung unggah bukti jika pembayaran manual.
- Notifikasi pengingat H-3, H-1, dan saat kadaluarsa.

### 5.2 Discovery & Profil Guru
- Filter: radius lokasi, rentang harga, kurikulum, grade, preferensi lokasi (home/studio/online), ketersediaan slot, rating.
- Kartu guru menampilkan foto, nama, badge verifikasi, harga per durasi, jarak, badge kurikulum.
- Profil detail: bio, kredensial, pengalaman, jadwal, kebijakan pembatalan, biaya transport, peta lokasi, review, rekomendasi buku.
- Aksi: simpan ke favorit, bagikan, chat (butuh pass aktif), pesan sesi.

### 5.3 Booking & Penjadwalan Les
- Student memilih tanggal, waktu, durasi (30/45/60/90 menit), tipe lokasi, alamat (bila guru ke rumah).
- Guru menindaklanjuti: `Accept`, `Reschedule` (usulkan slot baru), `Decline`.
- Pembayaran sesi:
  - **Pra bayar**: wajib lunas sebelum sesi dimulai.
  - **Pasca bayar**: jika guru mengizinkan, dengan tenggat pembayaran otomatis (misal 24 jam setelah sesi selesai).
- Status booking: `Requested → Accepted → Paid/Unpaid → In Transit → In Session → Completed → Rated`. Cabang: `Rescheduled`, `Cancelled`, `No Show`, `Dispute`.
- Konflik jadwal dicegah sistem (double booking / overlap slot).

### 5.4 Pembayaran & Komisi
- Instruksi pembayaran unik per transaksi (VA nomor unik atau kode transfer).
- Bukti transfer dapat diunggah; verifikasi manual/otomatis.
- Komisi platform untuk les berupa persentase configurable, dipotong saat payout ke guru.
- Invoice & struk tersedia di modul Pembayaran.

### 5.5 Verifikasi Kehadiran
- Guru menekan **Start Session**: aplikasi mengambil GPS + foto dengan overlay timestamp (wajah guru + siswa/instrumen sesuai kebijakan). Radius default 200 m dari lokasi yang disepakati, configurable.
- **Finish Session**: capture koordinat + foto opsional.
- Data disimpan sebagai `Attendance`. Dukungan offline queue: jika koneksi buruk, data disinkron saat online.
- Sistem memberi peringatan bila start di luar radius; guru dapat memasukkan alasan (flag untuk admin review).

### 5.6 Progress & Kolaborasi Orang Tua
- Guru wajib mengisi form progress ≤24 jam setelah sesi: topik, teknik, lagu, PR, target latihan (menit), tujuan berikutnya.
- Lampiran pendukung: audio/video pendek, foto catatan.
- Orang tua menerima notifikasi saat progress dipublish dan dapat membalas via komentar thread per sesi.

### 5.7 Rating & Review
- Student/Parent memberi rating 1–5 + komentar setelah sesi selesai.
- Guru memberikan rating reliabilitas murid (hanya terlihat oleh admin untuk monitoring).
- Aturan anti-spam: satu rating per booking; bila Student & Parent keduanya rating, skor digabung (misal rata-rata) atau ikuti kebijakan produk.

### 5.8 Marketplace Buku (Checkout In-App)
- Seller mendaftarkan produk: judul, gambar, level/grade, deskripsi, harga, SKU, varian, stok, bobot, opsi ekspedisi & ongkir.
- Pengguna (Student/Parent) menambah ke keranjang, pilih alamat & kurir, bayar via transfer/VA (tidak ada tautan keluar aplikasi).
- Biaya platform (per order) otomatis dihitung:
  - `< IDR 100.000` → potongan **IDR 3.000**
  - `IDR 100.000 – 299.999` → potongan **IDR 5.000**
  - `≥ IDR 300.000` → potongan **IDR 7.000**
- Alur order: `Dibayar → Menunggu Diproses → Dikirim (input resi) → Diterima → Selesai`. Jalur sengketa/refund aktif bila diperlukan.
- Payout ke seller = total dibayar − biaya platform − ongkir (ongkir diteruskan sesuai kebijakan kurir).
- Guru dapat menandai buku rekomendasi; murid menerima notifikasi & shortcut checkout.

### 5.9 Chat In-App
- Chat terbuka antara Student/Parent ↔ Guru setelah pass aktif atau ada booking pending.
- Fitur: pesan teks, gambar, PDF, template cepat (alamat, reschedule, tips latihan, rekomendasi buku).
- Chat disimpan ter-enkripsi, tersedia untuk referensi sengketa bila diperlukan.

### 5.10 Sengketa & Bantuan
- Pengguna dapat membuka kasus untuk: `No Show`, kualitas pengajaran, masalah pembayaran, pesanan buku (telat, produk salah/ rusak).
- Upload bukti (foto, dokumen), timeline penanganan ditampilkan.
- Admin mengelola status sengketa hingga resolusi (refund, kredit, dll.).

---

## 6. Field & Data Model Utama (High Level)

```text
User { id, phone, email, roles[], status }
Profile { user_id, name, photo, bio }
ParentLink { parent_id, student_id }
StudentPass { user_id, status, start_at, end_at, txn_id }
Teacher { user_id, grades[], curricula[], base_location{lat,lng}, travel_pref, travel_radius_km,
          travel_fee_rules, session_prices[], availability_rules, cancel_policy, kyc_status, payout_bank }
TeacherDocument { teacher_id, type, file_url, verified }
Booking { id, student_id, teacher_id, start_time, end_time, duration, location_type, address,
          status, price, travel_fee, total_due, payment_status }
Attendance { booking_id, start{lat,lng,time,photo_url}, end{lat,lng,time,photo_url} }
ProgressLog { booking_id, topics, notes, homework, targets, media_urls[], parent_comments[] }
Payment { id, type(pass|lesson|book), ref_id, method, amount, bank_ref, proof_url, status }
Review { booking_id, rater_id, stars, comment, created_at }
Seller { user_id, shop_name, kyc_status, payout_bank }
BookListing { seller_id, sku, title, level, price, image_url, description, stock, weight,
              variants[], shipping_options[], is_active }
Cart { user_id, items[{ listing_id, variant_id?, qty, price_snapshot }] }
BookOrder { id, user_id, seller_id, items[], subtotal, shipping_fee, total_paid,
            fee_tier_applied, fee_amount, payout_amount, address, courier, airwaybill, status }
Referral { teacher_id?, listing_id?, clicks, orders }
Dispute { ref_type(booking|order), ref_id, opener_id, reason, evidence[], status, resolution }
Notification { user_id, type, payload, read }
```

> Pastikan semua timestamp disimpan sebagai UTC di backend tetapi ditampilkan sebagai WIB pada UI. Audit trail diwajibkan untuk setiap perubahan status entitas kritis (Booking, Payment, BookOrder, Dispute).

---

## 7. Alur Kunci End-to-End

### 7.1 Pembelian Student Pass
1. Student memilih paket RIA Access (IDR 59.900) → menerima instruksi transfer/VA.
2. Jika pembayaran manual, unggah bukti → admin/otomasi memverifikasi → status pass `Active` + set `start_at` & `end_at` (30 hari).
3. Notifikasi sukses dikirim ke Student & Parent (jika tertaut).

### 7.2 Booking Les
1. Student membuka profil guru → tekan **Pesan**.
2. Pilih slot (tanggal, jam, durasi), tipe lokasi, alamat bila guru ke rumah.
3. Sistem menghitung total (harga sesi + biaya transport) dan menampilkan ringkasan.
4. Student kirim permintaan; Guru menerima & pilih `Accept/Reschedule/Decline`.
5. Jika pra bayar → Student melakukan pembayaran; status booking `Paid`. Jika pasca bayar → status `Unpaid` dengan tenggat.
6. Hari H: Guru menekan **Start Session** (GPS + foto) → status `In Session`.
7. Sesi selesai: Guru menekan **Finish Session** (opsional foto) → status `Completed`.
8. Guru mengisi Progress log ≤24 jam → notifikasi ke Student & Parent → Student/Parent memberi rating.

### 7.3 Verifikasi Kehadiran (Detail)
- Validasi radius default 200 m (konfigurabel). Bila melebihi, munculkan peringatan & minta alasan; flag untuk admin audit.
- Foto wajib jelas (sistem bisa memvalidasi ukuran & blur, minimal resolusi tertentu). Re-upload diperbolehkan ≤12 jam jika foto buram.
- Mode offline: data disimpan lokal dan disinkron begitu koneksi tersedia; status booking menunggu sinkronisasi.

### 7.4 Progress & Komentar Orang Tua
- Form progress memiliki autosave per field (topik, teknik, lagu, PR, target latihan menit, tujuan berikutnya).
- Orang tua dapat meninggalkan komentar (thread) untuk klarifikasi; guru dapat membalas. Semua percakapan tercatat.

### 7.5 Belanja Buku (Checkout In-App)
1. Pengguna menjelajah marketplace atau membuka rekomendasi guru.
2. Tambah ke keranjang → lanjut checkout (alamat + pilihan kurir + ongkir).
3. Sistem menghitung total + biaya platform tier (lihat §5.8) sebelum konfirmasi pembayaran.
4. Pembayaran transfer/VA → verifikasi → status order `Dibayar`.
5. Seller menerima order, menyiapkan barang, meng-input resi saat kirim → status `Dikirim`.
6. Pengguna konfirmasi penerimaan → status `Selesai`; sistem menjadwalkan payout ke seller = `total_paid - fee_amount - shipping_fee_pass_through`.
7. Jika masalah (barang rusak/salah/telat) → pengguna membuka sengketa; admin menentukan resolusi (refund, kirim ulang, dsb.).

---

## 8. Aturan Harga, Biaya & Payout

- **Harga sesi**: ditentukan guru per durasi (30/45/60/90 menit). Sistem mendukung tambahan biaya transport (flat atau bertahap).
- **Komisi platform les**: persentase configurable, dipotong dari penghasilan guru saat payout.
- **Student Pass**: IDR 59.900 (default) untuk 30 hari; admin dapat mengubah harga & durasi di console.
- **Biaya marketplace** (flat per order, otomatis terapkan ke `BookOrder.fee_amount`):
  - `< 100.000` → IDR 3.000
  - `100.000 – 299.999` → IDR 5.000
  - `≥ 300.000` → IDR 7.000
- **Payout**: dijalankan mingguan atau sesuai jadwal, ke rekening yang diverifikasi pada profil guru/seller. Laporan rinci menampilkan breakdown transaksi, komisi, biaya, ongkir, sengketa.
- **Pembatalan & Refund**:
  - Student batal melewati window yang diizinkan guru → fee kompensasi ke guru (tetap memperhitungkan komisi platform sesuai kebijakan).
  - Guru batal mendadak → kredit/refund ke student; admin dapat memberi penalti.
  - Pesanan buku dibatalkan sebelum pengiriman → refund penuh; setelah kirim → ikuti alur retur/sengketa.

**Rationale biaya marketplace**: nominal flat kecil agar kompetitif, mudah dipahami seller, serta cukup menutup biaya operasional pembayaran & dukungan pelanggan.

---

## 9. Edge Case Prioritas

1. Student tanpa pass aktif mencoba booking → blokir, tampilkan CTA beli pass.
2. Konflik jadwal guru (double booking) → sistem menolak pembuatan booking baru pada slot overlap.
3. Bukti transfer salah nominal → kirim notifikasi minta unggah ulang atau refund otomatis bila perlu.
4. Start session di luar radius → peringatan + input alasan; tandai untuk admin review.
5. Koneksi buruk saat start/finish → simpan offline & sinkron otomatis saat online.
6. Foto kehadiran buram → minta re-upload maksimal 12 jam setelah sesi.
7. Rating dari Student & Parent → gabungkan sesuai kebijakan (misal rata-rata) agar tidak menggandakan pengaruh.
8. Reschedule dalam window pembatalan → gunakan aturan terbaru (durasi cancel/reschedule update real-time dari kebijakan guru).
9. Marketplace: stok habis saat checkout → validasi ulang sebelum pembayaran; jika gagal, informasikan & update keranjang.
10. Marketplace: alamat tidak terjangkau kurir → minta ganti kurir atau edit alamat.
11. Marketplace: ongkir mismatch → validasi ulang sebelum order dikonfirmasi.

---

## 10. Notifikasi & Komunikasi

- Push & in-app notification untuk: perubahan status booking/order, verifikasi pembayaran, pengingat sesi (H-1 & H-0), progress terbit, pass hampir kadaluarsa, payout berhasil.
- Email/SMS sebagai fallback untuk event kritikal (verifikasi pembayaran, reset password, sengketa).
- Semua notifikasi disimpan di tabel `Notification` dengan status baca.

---

## 11. Keamanan & Kepatuhan

- KYC wajib untuk guru & seller (KTP + selfie; seller dapat butuh legalitas toko).
- Verifikasi kehadiran kombinasi GPS + foto bertimestamp (simpan metadata EXIF bila ada).
- Deteksi anomali otomatis: start jauh dari lokasi, waktu tempuh tidak wajar, pola refund mencurigakan.
- Laporan & pelaporan pengguna tersedia; admin dapat menonaktifkan akun.
- Penyimpanan foto pada object storage dengan akses terkontrol; generate thumbnail aman untuk tampilan orang tua.
- Logging & audit trail untuk seluruh perubahan status penting.

---

## 12. Admin Console (Fitur v1.1)

- **Dashboard** agregat metrik pass, booking, marketplace.
- **Users & KYC**: verifikasi dokumen, beri badge guru tersertifikasi.
- **Teachers**: kelola profil, jadwal, suspend/activate.
- **Bookings**: lihat timeline, override status bila sengketa selesai.
- **Marketplace & Pesanan**: moderasi listing, pantau SLA pengiriman.
- **Pembayaran & Payout**: ledger transaksi, jadwal payout, konfirmasi manual bila diperlukan.
- **Sengketa**: antrian kasus dengan viewer bukti.
- **Moderasi Konten**: review, komentar progress, chat terlapor.
- **Analytics**: metrik versi v1.1 (konversi pass, completion rate sesi, dll.).
- **Konfigurasi**: set harga pass, komisi les, tier biaya marketplace, radius GPS, kebijakan refund/reschedule.

---

## 13. Non-Fungsional & Aksesibilitas

- Mobile-first; UI adaptif untuk berbagai ukuran layar.
- Aksesibilitas dasar: target sentuh ≥44px, kontras warna sesuai WCAG AA minimal.
- Dukungan lokaliasi Bahasa Indonesia & Inggris (EN).
- Performa: caching daftar guru/produk, pagination, penanganan offline queue untuk attendance & progress.
- Keamanan data pribadi (PDPA setempat), enkripsi data sensitif saat transit & rest.

---

## 14. Sketsa API (Referensi)

```
POST /auth/otp
POST /users

GET  /teachers?lat=&lng=&radius=&price_min=&price_max=&curricula=
GET  /teachers/{id}
POST /bookings { teacher_id, start_time, duration, location_type, address }
PATCH /bookings/{id} (accept|decline|reschedule)
POST /payments/transfer { type: pass|lesson|book, ref_id, amount, proof_url }
POST /attendance/start { booking_id, lat, lng, photo }
POST /attendance/end { booking_id, lat, lng, photo }
POST /progress { booking_id, fields }
GET  /progress?student_id=

POST /listings { seller_id, sku, title, price, stock, weight, shipping_options, image_url }
GET  /listings?level=&q=&sort=
POST /cart/items { listing_id, variant_id?, qty }
POST /checkout { cart_id, address, courier }
POST /orders/{id}/ship { airwaybill }
GET  /orders?role=seller|buyer&status=
```

Endpoint final dapat disesuaikan, namun struktur ini cukup untuk menurunkan kontrak API awal.

---

## 15. Roadmap Singkat

- **v1.0**: Student Pass, discovery guru, booking + transfer manual, attendance dasar, progress, marketplace checkout, review, admin minimal.
- **v1.1 (dokumen ini)**: VA auto-verifikasi, tier biaya marketplace final (3k/5k/7k), payout dashboard, komentar orang tua pada progress.
- **v1.2**: Analytics lanjutan, promo/voucher, ongkir real-time dari partner logistik.

---

## 16. Metrik Keberhasilan

- Konversi Student Pass (visitor → pembeli pass) & waktu ke booking pertama.
- Tingkat penyelesaian sesi & ketepatan waktu start.
- Kepatuhan pengisian progress ≤24 jam.
- MAU orang tua, retensi guru, rata-rata penghasilan guru.
- Marketplace: conversion rate keranjang → bayar, SLA pengiriman, persentase sengketa.

---

## 17. Lampiran: Sumber Desain & Wireframe

- Aksi utama (Pesan, Bayar, Start, Finish, Submit, Checkout) ditempatkan di bagian bawah layar.
- Gunakan kartu (card) untuk profil guru, progress log, dan listing marketplace.
- Sertakan mini map pada profil guru & halaman booking lokasi.
- Form progress bertahap dengan autosave; tampilkan status penyimpanan.
- Marketplace memiliki tab: **Produk**, **Keranjang**, **Pesanan**.

---

**Dokumen ini berlaku sebagai baseline implementasi PianoRIA v1.1. Perubahan bisnis di luar ruang lingkup harus melalui proses change request.**
