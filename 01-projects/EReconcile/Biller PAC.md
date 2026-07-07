
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
  values ('source_account', 'debit_account_id', 'acquirer', 'card_no', 'fee_admin', 'recvbic', 'debtbic', 'partner_reference', 'status', 'product_name', 'settlement_date', 'fee_admin', 'journal_reference_id', 'transaction_date', 'transaction_type', gmparameter0_id, 9, 4, 2, 1, 9, 4, 4, 9, 4, 9, 9, 9, 9, 9, 1)

-- 2. Create gmparameter4


-- 3. Insert gmfiletype1
INSERT INTO "bina-erecon"."public"."gmfiletype1" (lineno,idno,field,mappingtodatamart,startindex,endindex,row,changeno,createby,createdate,changeby,changedate,linenomappingdatamart) VALUES
(gmfiletype0_id,1,'RRN','partner_reference',NULL,NULL,NULL,'0','System',NOW(),'System',NOW(),9),
(gmfiletype0_id,2,'DATE','settlement_date',NULL,NULL,NULL,'0','System',NOW(),'System',NOW(),9),
(gmfiletype0_id,3,'TIME','transaction_date',NULL,NULL,NULL,'0','System',NOW(),'System',NOW(),9),
(gmfiletype0_id,4,'NOTELP','',NULL,NULL,NULL,'0','System',NOW(),'System',NOW(),NULL),
(gmfiletype0_id,5,'AMOUNT','total_amount',NULL,NULL,NULL,'0','System',NOW(),'System',NOW(),9);

-- 4. Update filter 
UPDATE gmfilterreconciliation0 
SET ibmb = '0', biller = '1' 
WHERE linenoparameter = gmparameter0_id;



```