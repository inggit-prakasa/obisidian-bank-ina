### 2.1 Arsitektur Alur Data

```
[Operation Portal - MySQL]
   table: transaction (t) JOIN payment (p)
        │
        │ extract (filter: provider_code='OTTOAG', type IN PAY0001/PUR0001, status<>REVERSAL, date=business_date)
        ▼
[Airflow Task: etl_biller_transaction_summary]
        │
        │ executemany INSERT
        ▼
[erecon - PostgreSQL: TEMP TABLE temp_stg_digital_otto]
        │
        │ INNER JOIN dengan tabel mutation (match: external_id = no_reference, value_date = business_date)
        ▼
[erecon - PostgreSQL: biller_transaction_summary]  (final target)
```

### 2.2 Detail Komponen Teknis

| Komponen                 | Nilai                                           |
| ------------------------ | ----------------------------------------------- |
| DAG ID                   | `erecon_ibmb_transaction_etl_pipeline`          |
| Schedule                 | `0 22 * * *` (22:00 UTC = 05:00 WIB)            |
| Start Date               | 2026-05-05                                      |
| Catchup                  | `False`                                         |
| Retries                  | 2, dengan `retry_delay` 5 menit                 |
| Source Connection        | `db_operation_portal` (MySQL, via `MySqlHook`)  |
| Target Connection        | `db_erecon` (PostgreSQL, via `PostgresHook`)    |
| Parameter manual trigger | `business_date` (format `YYYY-MM-DD`, opsional) |

### 2.3 Logika Penentuan `business_date`

Fungsi `resolve_business_date()`:

1. Jika parameter `business_date` diisi saat trigger manual → gunakan nilai tersebut.
2. Jika kosong (scheduled run) → hitung otomatis: waktu UTC saat ini + 7 jam (WIB) dikurangi 1 hari.
3. Log dicatat untuk kedua kondisi (manual vs otomatis) guna keperluan audit run.

### 2.4 Spesifikasi Ekstraksi (Source: MySQL)

**Query:** `SELECT` dari `transaction t LEFT JOIN payment p ON p.id = t.payment_id`

**Filter:**

- `t.provider_code = 'OTTOAG'`
- `t.transaction_type_code IN ('PAY0001', 'PUR0001')`
- `t.status != 'REVERSAL'`
- `DATE(t.transaction_time) = business_date`

**Kolom yang diambil:**

|Kolom Sumber|Keterangan|
|---|---|
|`t.account_id`|ID akun sumber transaksi|
|`t.status`|Status transaksi|
|`t.provider_code`|Kode provider (OTTOAG)|
|`t.no_reference`|Nomor referensi internal|
|`t.transaction_time`|Waktu transaksi|
|`p.destination_id`|ID tujuan pembayaran|
|`p.provider_reference_no`|Nomor referensi dari provider/partner|
|`p.product_name`|Nama produk|
|`p.basic_amount`|Nominal dasar|
|`p.admin_fee`|Biaya admin|
|`p.total_amount`|Total nominal|

### 2.5 Spesifikasi Staging (Target Sementara: PostgreSQL)

**Tabel:** `temp_stg_digital_otto` (TEMP TABLE, lifecycle per-session)

|Kolom|Tipe|
|---|---|
|`account_id`|VARCHAR(30)|
|`transaction_status`|VARCHAR(20)|
|`provider_code`|VARCHAR(20)|
|`no_reference`|VARCHAR(50)|
|`transaction_time`|TIMESTAMP|
|`destination_id`|VARCHAR(50)|
|`provider_reference_no`|VARCHAR(50)|
|`product_name`|VARCHAR(255)|
|`basic_amount`|DECIMAL(15,2)|
|`admin_fee`|DECIMAL(15,2)|
|`total_amount`|DECIMAL(15,2)|

Insert dilakukan dengan `cursor.executemany()` dari hasil `records` ekstraksi MySQL.

### 2.6 Spesifikasi Load Final (Target: `biller_transaction_summary`)

**Join Condition:** `temp_stg_digital_otto.no_reference = mutation.external_id AND mutation.value_date = business_date`

**Mapping Field:**

|Kolom Target|Sumber / Nilai|
|---|---|
|`journal_source`|Konstanta `'DIGITAL'`|
|`journal_reference_id`|`mutation.external_id`|
|`internal_reference`|`temp.no_reference`|
|`partner_reference`|`temp.provider_reference_no`|
|`transaction_type`|Konstanta `'BILLER_OTTOPAY'`|
|`source_account`|`temp.account_id`|
|`destination_account`|`NULL`|
|`biller_code`|`temp.provider_code`|
|`customer_reference`|`temp.destination_id`|
|`product_name`|`temp.product_name`|
|`base_amount`|`temp.basic_amount`|
|`fee_admin`|`temp.admin_fee`|
|`total_amount`|`temp.total_amount`|
|`transaction_date`|`temp.transaction_time`|
|`settlement_date`|`mutation.value_date`|
|`transaction_status`|`temp.transaction_status`|
|`partner_code`|Konstanta `'OTTO'`|

Hanya baris yang berhasil `INNER JOIN` dengan `mutation` yang akan masuk ke tabel final (transaksi tanpa pasangan mutasi akan **hilang secara diam-diam**, lihat catatan gap di 2.8).

### 2.7 Error Handling & Retry

- Airflow `default_args`: retry 2x dengan jeda 5 menit apabila task gagal (exception di level koneksi/query).
- Tidak ada exception handling eksplisit (`try/except`) di dalam fungsi — kegagalan query akan membatalkan seluruh transaksi (rollback implisit karena `conn.commit()` belum tercapai).
- Koneksi ditutup di akhir (`conn.close()`), namun tidak dalam blok `finally`, sehingga bila terjadi error sebelum baris ini, koneksi berpotensi tidak tertutup dengan bersih.

### 2.8 Gap & Rekomendasi Perbaikan (Catatan Fungsional)

| #   | Temuan                                                                                                         | Rekomendasi                                                                                                         |
| --- | -------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| 1   | Tidak ada penanganan untuk transaksi yang gagal _match_ dengan `mutation` (INNER JOIN membuang unmatched rows) | Tambahkan LEFT JOIN + tabel/log "unmatched transactions" untuk investigasi harian                                   |
| 2   | Tidak ada `ON CONFLICT` / unique constraint saat insert ke `biller_transaction_summary`                        | Tambahkan upsert (`ON CONFLICT DO UPDATE/NOTHING`) agar re-run bersifat idempotent                                  |
| 3   | `conn.close()` tidak berada dalam `try/finally`                                                                | Bungkus proses DB dengan `try/finally` agar koneksi selalu tertutup                                                 |
| 4   | Tidak ada logging jumlah baris extracted vs loaded                                                             | Tambahkan `logging.info` untuk jumlah `records`, jumlah insert ke temp, dan jumlah insert final untuk observability |
| 5   | Status `REVERSAL` dikecualikan sepenuhnya, tanpa pencatatan                                                    | Pertimbangkan apakah reversal perlu direkam sebagai koreksi di summary, sesuai kebutuhan bisnis rekonsiliasi        |
