
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

1. Cause funds_transfer and teller is already to database bina-erecon, we should to get all data ibmb from bigquery
	```
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
	```
2. Mapping data big query to table ibmb_transaction_summary

	```
```