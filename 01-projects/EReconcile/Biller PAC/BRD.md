# Business Requirements Document (BRD)

## Project Name: E-Reconciliation Biller Transaction (PAC) ETL Pipeline
**Document Version:** 1.0  
**Status:** Draft  
**Target File:** [etl_erecon_biller_transaction_pac.py](../etl_erecon_biller_transaction_pac.py)  
**Date:** 2026-07-21  

---
## 1. Executive Summary

### 1.1 Business Background
Bank INA (Bankina) collaborates with third-party biller aggregators to facilitate utility, bill payment, and reload transactions for its customers across various channels, including Internet Banking & Mobile Banking (IBMB), Teller services, and automated Funds Transfers. One of the core biller aggregator partners is **PAC** (PT Point Alternatif Cerdas).

To ensure financial accuracy, identify discrepancies, and perform daily settlement with PAC, the finance and operations team requires a daily e-reconciliation (e-recon) process. This process compares transactions recorded on the bank's channels against those reported by PAC.

### 1.2 Purpose of the Document
This document defines the business requirements for the ETL (Extract, Transform, Load) pipeline orchestrating the aggregation of PAC-related transactions. This data is extracted from the bank's core banking system (T24 DWH tables) and digital channels, consolidated, and structured into a central reconciliation schema.

---

## 2. Business Objectives

* **Data Consolidation:** Automate the daily collection of all transaction data involving the PAC biller partner from diverse source systems (BigQuery e-banking tables, T24 Core Banking Postgres tables).
* **Accurate Financial Attributes:** Ensure the transaction base amount, administrative fee, and total transaction amount are precisely separated and captured to compute partner settlements.
* **Unified Reconciliation Source:** Load the formatted transaction summaries into the e-reconciliation database (`db_erecon`), specifically targeting the `ibmb_transaction_summary` and `biller_transaction_summary` tables, to serve as the single source of truth for daily matching.
* **T+1 Execution:** Ensure the pipeline completes daily transactions processing for the previous business day before core reconciliation runs.

---

## 3. Scope of Work

### 3.1 In-Scope
* **Channel Integrations:**
  * **IBMB (Internet Banking / Mobile Banking):** Transactions originating from PAC-connected billers.
  * **TELLER:** Manual over-the-counter payments processed under transaction code `210`.
  * **FUNDS_TRANSFER:** Automatic or customer-initiated funds transfers with transaction type `ACBL` or linked to IBMB receipts.
* **Calculations & Adjustments:**
  * Separating transaction base amounts and admin fees.
  * Special handling of calculations for specific utility products (e.g., PDAM Surabaya, PDAM Palyja, PDAM Aetra Tangerang).
* **Consolidations & Deduplication:**
  * Transforming and mapping disparate data models into standard Target tables.
  * Deduplication logic to prevent double counting of records using composite unique keys.

### 3.2 Out-of-Scope
* **Reconciliation Processing:** The actual comparison logic, dispute resolution flow, and automated matching reports with PAC.
* **Partner File Delivery:** Delivery of reconciliation files to PAC (managed by dedicated SFTP job DAGs).

---

## 4. Key Business Rules and Logic

### 4.1 Data Sources
The ETL pipeline fetches transaction records from two main data environments:
1. **Google Cloud BigQuery (Digital Banking Store):** Contains detailed transaction-level data for Internet/Mobile Banking transactions (`ebanking_t_transaction`).
2. **PostgreSQL E-Recon Database (T24 Data Mirror):** Contains local mirrors of Core Banking tables:
   * `"TELLER"`: Records cash transactions at bank branches.
   * `"FUNDS_TRANSFER"`: Records inter-account and electronic transfers.

### 4.2 Target Database
All consolidated data is loaded into the Postgres target database `db_erecon` in two primary staging tables:
1. `public.ibmb_transaction_summary`: Intermediary table storage for processed IBMB records.
2. `public.biller_transaction_summary`: The final unified target table for all channels (Teller, Funds Transfer, and IBMB) to be reconciled.

### 4.3 Transaction Mapping Rules by Channel

#### A. Internet Banking / Mobile Banking (IBMB)
* **Target:** `ibmb_transaction_summary`
* **Filter:** Extract records for the given business date with specific transaction translation codes representing billing/payments.
* **Partner Identification:** Hardcoded `partner_code = 'PAC'` and `journal_source = 'IBMB'`.
* **Amount and Fee Separation:**
  * Generally: `total_amount = transaction_amount + fee`.
  * Special Cases for Specific Billers (due to data format differences):
    * **PDAM Surabaya:** Base amount is parsed from raw JSON transaction data (`amount` key).
    * **PDAM Palyja / Aetra Tangerang:** Base amount is calculated as `premiumAmount + penalty` from raw JSON transaction data.
    * **Q1 (Unsuccessful Transaction):** Base amount is parsed as `transaction_amount - fee`.

#### B. Conventional Teller Transactions
* **Target:** `biller_transaction_summary`
* **Filter:** Transaction Code (`transaction_code`) = `'210'` and Business Date = target business date.
* **Partner Identification:** Hardcoded `partner_code = 'PAC'` and `journal_source = 'CONVENTIONAL'`, `channel_id = 'TELLER'`.
* **Amount Mapping:**
  * `total_amount` = `amount_local_1`.
  * `fee_admin` = `chrg_amt_local` (default to 0 if empty).
  * `base_amount` = `amount_local_1 - chrg_amt_local`.

#### C. Conventional Funds Transfer Transactions (Via IBMB Join)
* **Target:** `biller_transaction_summary`
* **Filter:** Joins Core Banking `"FUNDS_TRANSFER"` with `ibmb_transaction_summary` for the business date.
* **Logic:** Match on receipt number to partner reference, debit account to source account, and credit amount to base amount.
* **Partner Identification:** Hardcoded `partner_code = 'PAC'`, `journal_source = 'CONVENTIONAL'`, `channel_id = 'FUNDS_TRANSFER'`.

#### D. Conventional Funds Transfer Transactions (Direct ACBL)
* **Target:** `biller_transaction_summary`
* **Filter:** Transaction Type (`transaction_type`) = `'ACBL'` and Business Date = target business date.
* **Partner Identification:** Hardcoded `partner_code = 'PAC'`, `journal_source = 'CONVENTIONAL'`, `channel_id = 'FUNDS_TRANSFER'`.
* **Amount Mapping:**
  * `total_amount` = `debit_amount` or `credit_amount`.
  * `fee_admin` = `local_charge_amt`.
  * `base_amount` = `total_amount - fee_admin`.

---

## 5. Non-Functional Requirements

### 5.1 Data Integrity & Deduplication
* Duplicate transaction records must be strictly rejected. If a transaction with the same reference keys already exists in the target table, the new record must be ignored (`ON CONFLICT DO NOTHING`).
* No data loss: Dropped records (due to invalid formats, unparseable dates, or missing IDs) must be logged with warning levels for administrative visibility.

### 5.2 Performance and Scalability
* Extraction and loading must handle high-volume transaction days (e.g., month-end salary and billing periods).
* Target inserts must be done in batches (e.g., bulk insert patterns) rather than row-by-row queries to minimize database locking and connection times.

### 5.3 Reliability and Operational Monitoring
* The pipeline must execute automatically at **22:00 UTC** daily.
* In the event of temporary database network cuts, the DAG should attempt to retry processing up to **2 times** with a **5-minute delay**.
* Support for manual execution is required, allowing operational staff to trigger specific runs by supplying a custom `business_date` (e.g. for backfilling or reprocessing failed days).
