
- Diagram =>  [[Biller PAC Diagram]]

Source : 
- **FUNDS_TRANSFER** (bina-erecon)
- **IBMB_ADMIN** (bigquery)
	```
  Select * from dina-prd-env.bina_pac_channel_ebanking.ebanking_t_transaction t
	Inner Join dina-prd-env.bina_pac_channel_ebanking.ebanking_m_customer c 
	  On c.id = t.m_customer_id 
	Left Join dina-prd-env.bina_pac_channel_ebanking.ebanking_t_transaction_data td
    ```
- **TELLER** (bina-erecon)

> ibmb_admin is data transaction ft or teller is journal transaction, so if  we reconcile must left join funds_transfer or teller first next to ibmb_admin

### Flow mapping data to biller_transaction_summary

1. Cause funds_transfer and teller is already to database **bina-erecon**, we should to get all data ibmb from bigQuery
	```sql
	SELECT  
	  t.id,  
	  t.transaction_date,  
	  t.delivery_channel,  
	  c.cif_number,  
	  c.customer_username,  
	  c.customer_name,  
	  t.reference_number,  
	  t.from_account_number,  
	  t.translation_code,  
	  t.customer_reference,  
	  CASE  
	    WHEN LOWER(t.translation_code) = 'q1' AND t.status != 'SUCCEED'  
	      THEN (t.transaction_amount - t.fee)  
	    WHEN LOWER(t.translation_code) = 'b4' AND t.biller_name IN ('PDAM Surabaya')  
	      THEN CAST(JSON_VALUE(td.transaction_data, '$.amount') AS NUMERIC)  
	    WHEN LOWER(t.translation_code) = 'b4' AND t.biller_name IN ('PDAM Palyja Jakarta', 'PDAM Aetra Tangerang')  
	      THEN (CAST(JSON_VALUE(td.transaction_data, '$.premiumAmount') AS NUMERIC) + CAST(JSON_VALUE(td.transaction_data, '$.penalty') AS NUMERIC))  
	    ELSE t.transaction_amount  
	  END AS transaction_amount,  
	  t.fee,  
	  t.response_code,  
	  t.biller_id,  
	  td.class_name,  
	  td.transaction_data,
	  CASE  
	    WHEN LOWER(t.translation_code) = 'b4'  
	      THEN JSON_VALUE(td.transaction_data, '$.billerReference')  
	    WHEN LOWER(t.translation_code) IN ('71', '73', '75', '77', 't2')  
	      THEN JSON_VALUE(td.transaction_data, '$.receiverName')  
	    ELSE t.free_data1  
	  END AS free_data1,
	  t.free_data2,  
	  t.free_data3,  
	  t.product_id,  
	  t.status,  
	  t.description,  
	  t.biller_name,  
	  t.ip_address
	 from dina-prd-env.bina_pac_channel_ebanking.ebanking_t_transaction t 
		 Inner Join dina-prd-env.bina_pac_channel_ebanking.ebanking_m_customer c 
	  On c.id = t.m_customer_id 
		  Left Join dina-prd-env.bina_pac_channel_ebanking.ebanking_t_transaction_data td
	  ON td.t_transaction_id = t.id

	```
2. Mapping data bigQuery to table **ibmb_transaction_summary**
	```sql
	
	```


```
Create a new ETL pipeline (Airflow DAG, Python) to move transaction data
from BigQuery into the Postgres table public.ibmb_transaction_summary.
Follow the structure/conventions of existing ETL pipelines in this repo
(check skill.md and other DAGs like erecon_ibmb_transaction_etl_pipeline
for coding style, error handling, and task naming conventions).

SOURCE DATA (BigQuery, project: dina-prd-env)
Run the following query, filtered by the DAG run's execution window
(t.transaction_date = business_date):

SELECT
  t.id,
  t.transaction_date,
  t.delivery_channel,
  c.cif_number,
  c.customer_username,
  c.customer_name,
  t.reference_number,
  t.from_account_number,
  t.translation_code,
  t.customer_reference,
  CASE
    WHEN LOWER(t.translation_code) = 'q1' AND t.status != 'SUCCEED'
      THEN (t.transaction_amount - t.fee)
    WHEN LOWER(t.translation_code) = 'b4' AND t.biller_name IN ('PDAM Surabaya')
      THEN CAST(JSON_VALUE(td.transaction_data, '$.amount') AS NUMERIC)
    WHEN LOWER(t.translation_code) = 'b4' AND t.biller_name IN ('PDAM Palyja Jakarta', 'PDAM Aetra Tangerang')
      THEN (CAST(JSON_VALUE(td.transaction_data, '$.premiumAmount') AS NUMERIC) + CAST(JSON_VALUE(td.transaction_data, '$.penalty') AS NUMERIC))
    ELSE t.transaction_amount
  END AS transaction_amount,
  t.fee,
  t.response_code,
  t.biller_id,
  td.class_name,
  td.transaction_data,
  CASE
    WHEN LOWER(t.translation_code) = 'b4'
      THEN JSON_VALUE(td.transaction_data, '$.billerReference')
    WHEN LOWER(t.translation_code) IN ('71', '73', '75', '77', 't2')
      THEN JSON_VALUE(td.transaction_data, '$.receiverName')
    ELSE t.free_data1
  END AS free_data1,
  t.free_data2,
  t.free_data3,
  t.product_id,
  t.status,
  t.description,
  t.biller_name,
  t.ip_address
FROM `dina-prd-env.bina_pac_channel_ebanking.ebanking_t_transaction` t
INNER JOIN `dina-prd-env.bina_pac_channel_ebanking.ebanking_m_customer` c
  ON c.id = t.m_customer_id
LEFT JOIN `dina-prd-env.bina_pac_channel_ebanking.ebanking_t_transaction_data` td
  ON td.t_transaction_id = t.id

TARGET SCHEMA (Postgres)
CREATE TABLE public.ibmb_transaction_summary (
  id bigserial PRIMARY KEY,
  journal_source varchar(20),
  journal_reference_id varchar(50),
  internal_reference varchar(50),
  partner_reference varchar(50),
  transaction_type varchar(30),
  source_account varchar(30),
  destination_account varchar(30),
  biller_code varchar(20),
  customer_reference varchar(50),
  base_amount numeric(15,2),
  fee_admin numeric(15,2),
  total_amount numeric(15,2),
  transaction_date timestamp without time zone,
  channel_id varchar(20),
  transaction_status varchar(20),
  product_name varchar(100),
  settlement_date date
);

MAPPING (transaction_amount and free_data1 are ALREADY final from the
BigQuery CASE WHEN logic above — do not recompute them in Python):
- journal_source        = constant 'IBMB'
- journal_reference_id  = reference_number
- internal_reference    = id
- partner_reference     = NULL
- transaction_type      = translation_code
- source_account        = from_account_number
- destination_account   = NULL
- biller_code           = biller_id (convert empty string to NULL)
- customer_reference    = customer_reference
- base_amount           = transaction_amount
- fee_admin             = fee
- total_amount          = transaction_amount + fee
- transaction_date      = transaction_date
- channel_id            = delivery_channel
- transaction_status    = status
- product_name          = biller_name
- settlement_date       = DATE(transaction_date)
Unused BigQuery columns: cif_number, customer_username, customer_name,
response_code, class_name, transaction_data, free_data1, free_data2,
free_data3, product_id, description, ip_address.

REQUIREMENTS
- Add retries and clear logging in each task (rows in / dropped / errors).
- Never TRUNCATE the target table.
- schedule_interval @daily, catchup=False.
- Add clear comments explaining the business logic (Q1 refund adjustment,
  B4 PDAM override) so it stays maintainable.

Show me the full code before running it.
```