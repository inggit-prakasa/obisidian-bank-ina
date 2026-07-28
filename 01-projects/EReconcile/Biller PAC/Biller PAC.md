
- Diagram =>  [[Biller PAC Diagram]]

Source : 
- **FUNDS_TRANSFER** (bina-erecon)
- **IBMB_ADMIN** (bigquery)
	- Transaksi : 
		- Internet (88)
		- Pembayaran PLN (81)
		- PDAM (80)

	```
  Select * from dina-prd-env.bina_pac_channel_ebanking.ebanking_t_transaction t
	Inner Join dina-prd-env.bina_pac_channel_ebanking.ebanking_m_customer c 
	  On c.id = t.m_customer_id 
	Left Join dina-prd-env.bina_pac_channel_ebanking.ebanking_t_transaction_data td
    ```
	- filter by translation_code = 80,81,88
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



### Header for External Data

```txt
DATE|TIME|MERCHANT_CODE|AIIC|STAN|RRN|NOTELP|Param2|Param3|Param4|AMOUNT|ADMIN|BILLNUMBER|Ref1|Ref2|Ref3|Ref4|RECORD ID|TERMINAL ID|BILLER ID|DEALER ID
```



## Deployment Pre-requisite
### Service Deployment
- [extract-data-worker](https://gitlab.gcp.bankina.id/ina-ereconciliation/itk.ereconcile.extractdata.worker)
- [service-otomatis](https://gitlab.gcp.bankina.id/ina-ereconciliation/itk.ereconcile.service.otomatis)
### Query Execute

```sql

-- 0. Select gmparameter0 and gmfiletype0 with partipant is pac and channel biller
select * from gmparameter0 where channelname = 'Biller' and thirdpartyname = 'PAC'; -- 54
select * from gmfiletype0 where channel = 'Biller' and participant = 'PAC'; -- 58

-- 1. Create gmmappingreportDOP
INSERT INTO gmmappingreportdop (parameterid,sourcetransactionid,fieldtransactionid,sourcetransactiondate,fieldtransactiondate,sourcetransactiontime,fieldtransactiontime,sourceacquirerid,fieldacquirerid,sourceissuerid,fieldissuerid,sourceterminalid,fieldterminalid,sourcedestination,fielddestination,sourcereciptnumber,fieldreciptnumber,sourcetransactionfee,fieldtransactionfee,sourceaccountissuer,fieldaccountissuer,sourceaccountdestination,fieldaccountdestination,sourcetransactiontype,fieldtransactiontype,sourcechargeamount,fieldchargeamount,sourcecardnumber,fieldcardnumber,sourcestatus,fieldstatus) VALUES (gmparameter0_id,9,'journal_reference_id',9,'settlement_date',9,'transaction_date',2,'acquirer',4,'debtbic',9,'product_name',4,'recvbic',9,'partner_reference',9,'fee_admin',9,'source_account',2,'dest_acc_number',1,'transaction_type',9,'fee_admin',1,'card_no',4,'status');

-- 2. Create gmparameter4
delete from gmparameter4 where lineno = gmparameter0_id;

INSERT INTO gmparameter4 (lineno,idno,internaldata,externaldata,formulacolumn1,rules,formulacolumn2,tolerance,changeno,createby,createdate,changeby,changedate,flagamount) VALUES 
(gmparameter0_id,1,9,gmfiletype0_id,'(partner_reference)','=','(RRN)','',0,'System',NOW(),'System',NOW(),0),
(gmparameter0_id,2,9,gmfiletype0_id,'(base_amount)','=','(AMOUNT)','',0,'System',NOW(),'System',NOW(),1),
(gmparameter0_id,3,9,gmfiletype0_id,'(settlement_date)','=','(DATE)','1',0,'System',NOW(),'System',NOW(),0),
(gmparameter0_id,4,9,gmfiletype0_id,'(base_amount)','=','(AMOUNT)','',0,'System',NOW(),'System',NOW(),0);

-- 3. Insert gmfiletype1
delete from gmfiletype1 where lineno = gmfiletype0_id;

INSERT INTO gmfiletype1 (lineno,idno,field,mappingtodatamart,startindex,endindex,row,changeno,createby,createdate,changeby,changedate,linenomappingdatamart) VALUES
(gmfiletype0_id,1,'RRN','partner_reference',NULL,NULL,NULL,0,'System',NOW(),'System',NOW(),9),
(gmfiletype0_id,2,'DATE','settlement_date',NULL,NULL,NULL,0,'System',NOW(),'System',NOW(),9),
(gmfiletype0_id,3,'TIME','transaction_date',NULL,NULL,NULL,0,'System',NOW(),'System',NOW(),9),
(gmfiletype0_id,4,'NOTELP','',NULL,NULL,NULL,0,'System',NOW(),'System',NOW(),NULL),
(gmfiletype0_id,5,'AMOUNT','total_amount',NULL,NULL,NULL,0,'System',NOW(),'System',NOW(),9);

-- 4. Update filter 
UPDATE gmfilterreconciliation0 
SET ibmb = '0', biller = '1' 
WHERE linenoparameter = gmparameter0_id;

```