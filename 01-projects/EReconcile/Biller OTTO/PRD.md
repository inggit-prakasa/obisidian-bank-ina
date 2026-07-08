### 1.1 Latar Belakang

Sistem e-reconciliation (erecon) Bankina membutuhkan data transaksi biller dari kanal digital (PPOB/agen) untuk direkonsiliasi dengan data mutasi rekening (core banking). Salah satu partner biller yang perlu diintegrasikan adalah **OTTO/OTTOPAY (OTTOAG)**, yang transaksinya tercatat di _Operation Portal_ (MySQL) dan perlu digabungkan ke tabel konsolidasi `biller_transaction_summary` di database erecon (PostgreSQL).

Tanpa pipeline otomatis, proses rekonsiliasi transaksi OTTO harus dilakukan manual, yang berisiko:
- Keterlambatan proses rekonsiliasi harian.
- Selisih (mismatch) antara catatan biller dan mutasi rekening tidak terdeteksi tepat waktu.
- Human error dalam pencocokan data.

### 1.2 Tujuan (Objectives)

1. Mengotomatisasi ekstraksi transaksi OTTO dari Operation Portal setiap hari.
2. Melakukan pencocokan (matching) transaksi OTTO dengan data mutasi rekening (`mutation`) berdasarkan `external_id` dan `value_date`.
3. Menyimpan hasil transaksi yang berhasil dicocokkan ke tabel konsolidasi `biller_transaction_summary` sebagai sumber data rekonsiliasi lintas biller.
4. Menyediakan mekanisme _manual trigger_ dengan parameter `business_date` untuk keperluan re-run/backfill.

### 1.3 Target Pengguna (Stakeholders)

|Peran|Kebutuhan|
|---|---|
|Tim Reconciliation/Finance Ops|Data transaksi OTTO harian yang akurat & tepat waktu untuk proses rekon|
|Tim Data Engineering|Pipeline yang mudah dipantau, di-retry, dan di-backfill|
|Tim Audit/Kepatuhan|Jejak data (traceability) antara sumber biller dan mutasi rekening|

### 1.4 Ruang Lingkup (Scope)

**In-scope:**

- Ekstraksi data transaksi dengan `provider_code = 'OTTOAG'`, `transaction_type_code IN ('PAY0001','PUR0001')`, status bukan `REVERSAL`.
- Staging sementara di PostgreSQL (`temp_stg_digital_otto`).
- Join/matching dengan tabel `mutation` berbasis `external_id` dan `value_date`.
- Load hasil match ke `biller_transaction_summary`.
- Penjadwalan otomatis harian pukul 22:00 UTC (05:00 WIB).

**Out-of-scope (belum tercakup di kode saat ini):**

- Penanganan transaksi berstatus `REVERSAL`.
- Notifikasi/alerting saat jumlah transaksi hasil ekstraksi tidak match dengan hasil insert (selisih unmatched).
- Deduplikasi eksplisit sebelum insert ke tabel final (constraint dedup belum terlihat di query).
- Idempotency saat re-run (belum ada `ON CONFLICT` / `TRUNCATE` guard eksplisit pada `biller_transaction_summary`).

### 1.5 Kriteria Sukses (Success Metrics)

- Pipeline berjalan sukses (tanpa manual intervention) ≥ 99% dari total run harian.
- Selisih antara jumlah transaksi hasil ekstraksi dan jumlah baris yang berhasil di-load ke `biller_transaction_summary` termonitor dan dapat dijelaskan (root cause: unmatched mutation).
- Waktu eksekusi pipeline selesai sebelum jam kerja tim rekonsiliasi dimulai.

### 1.6 Asumsi

- Tabel `mutation` sudah tersedia dan terisi lebih dulu dari waktu jadwal DAG berjalan (05:00 WIB).
- Koneksi `db_erecon` (Postgres) dan `db_operation_portal` (MySQL) sudah dikonfigurasi di Airflow Connections.
- Satu `no_reference` di sisi biller berkorespondensi satu-ke-satu dengan satu `external_id` di tabel `mutation` untuk tanggal yang sama.

### 1.7 Risiko

| Risiko                                                 | Dampak                                                                   | Mitigasi yang Disarankan                      |
| ------------------------------------------------------ | ------------------------------------------------------------------------ | --------------------------------------------- |
| Transaksi tidak match dengan `mutation` (belum settle) | Transaksi hilang dari `biller_transaction_summary` tanpa jejak           | Tambahkan logging/tabel unmatched untuk audit |
| Re-run pipeline pada `business_date` yang sama         | Duplikasi data di `biller_transaction_summary` (belum ada `ON CONFLICT`) | Tambahkan unique constraint + upsert          |
| Volume data besar di `executemany`                     | Performa insert lambat                                                   | Pertimbangkan batch insert / `COPY`           |
|                                                        |                                                                          |                                               |
