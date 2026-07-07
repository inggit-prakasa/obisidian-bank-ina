# Source
## IBMB Big Query
- translation_code in OB and O8


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