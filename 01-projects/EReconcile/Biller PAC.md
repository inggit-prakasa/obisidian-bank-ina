
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

#### Mapping Funds Transfer with TRANSACTION_TYPE = ACBL




# Query Execute

```sql

-- 1. Create gmmappingreportDOP
insert into gmmappingreportdop ("fieldaccountdestination", "fieldaccountissuer", "fieldacquirerid", "fieldcardnumber", "fieldchargeamount", "fielddestination", "fieldissuerid", "fieldreciptnumber", "fieldstatus", "fieldterminalid", "fieldtransactiondate", "fieldtransactionfee", "fieldtransactionid", "fieldtransactiontime", "fieldtransactiontype", "parameterid", "sourceaccountdestination", "sourceaccountissuer", "sourceacquirerid", "sourcecardnumber", "sourcechargeamount", "sourcedestination", "sourceissuerid", "sourcereciptnumber", "sourcestatus", "sourceterminalid", "sourcetransactiondate", "sourcetransactionfee", "sourcetransactionid", "sourcetransactiontime", "sourcetransactiontype") 
  values ('source_account', 'debit_account_id', 'acquirer', 'card_no', 'fee_admin', 'recvbic', 'debtbic', 'partner_reference', 'status', 'product_name', 'settlement_date', 'fee_admin', 'journal_reference_id', 'transaction_date', 'transaction_type', , 9, 4, 2, 1, 9, 4, 4, 9, 4, 9, 9, 9, 9, 9, 1)

-- 6. Create gmparameter4
insert into gmparameter4 ("changeby", "changedate", "changeno", "createby", "createdate", "externaldata", "flagamount", "formulacolumn1", "formulacolumn2", "idno", "internaldata", "lineno", "rules", "tolerance") 
values 
('System', NOW(), '1', 'System', NOW(), '', 0, '(partner_reference)', '(App Ref No)', 2, 9, 33, '=', ''), 
('System', NOW(), '1', 'System', NOW(), '', 0, '(settlement_date)', '(Trx Date)', 3, 9, 33, '=', '1'), 
('System', NOW(), '1', 'System', NOW(), '', 1, '(total_amount)', '(Total Pay)', 1, 9, 33, '=', '')

-- 7. Insert gmfilterreconciliation2
insert into gmfilterreconciliation2 ("changeby", "changedate", "changeno", "createby", "createdate", "filterfield", "filtervalue", "idno", "linenoparameter", "sourcefileexternal") 
  values ('System', NOW(), '0', 'System', NOW(), 'Status', 'success', '1', 33, '')

-- 8. Insert gmfiletype1
insert into gmfiletype1 ("changeby", "changedate", "changeno", "createby", "createdate", "endindex", "field", "idno", "lineno", "linenomappingdatamart", "mappingtodatamart", "row", "startindex") 
values 
('System', NOW(), '0', 'System', NOW(), 2, 'Trx Date', '1', 37, '9', 'settlement_date', 1, 2), 
('System', NOW(), '0', 'System', NOW(), 3, 'Trx Time', '2', 37, '9', 'transaction_date', 1, 3), 
('System', NOW(), '0', 'System', NOW(), 4, 'App Ref No', '3', 37, '9', 'partner_reference', 1, 4), 
('System', NOW(), '0', 'System', NOW(), 5, 'Issuer Partner', '4', 37, NULL, '', 1, 5), 
('System', NOW(), '0', 'System', NOW(), 6, 'Jenis Biller', '5', 37, NULL, '', 1, 6), 
('System', NOW(), '0', 'System', NOW(), 7, 'Product Name', '6', 37, '9', 'product_name', 1, 7), 
('System', NOW(), '0', 'System', NOW(), 8, 'Account Number', '7', 37, '9', 'source_account', 1, 8), 
('System', NOW(), '0', 'System', NOW(), 9, 'Bill Amount', '8', 37, NULL, '', 1, 9), 
('System', NOW(), '0', 'System', NOW(), 12, 'Fee', '9', 37, '9', 'fee_admin', 1, 12), 
('System', NOW(), '0', 'System', NOW(), 13, 'Total Pay', '10', 37, '9', 'total_amount', 1, 13), 
('System', NOW(), '0', 'System', NOW(), 14, 'Status', '11', 37, '4', 'status', 1, 14

-- 9. Update filter 
UPDATE gmfilterreconciliation0 
SET ibmb = '0' 
WHERE linenoparameter = 52;

-- 2. tambah kolom biller dan set 1
ALTER TABLE gmfilterreconciliation0 ADD COLUMN biller varchar DEFAULT '0';

UPDATE gmfilterreconciliation0 
SET biller = '1' 
WHERE linenoparameter = 52;

-- select * from gmparameter0 where lineno = 52;
-- select * from gmfilterreconciliation0 where linenoparameter = 52;


```