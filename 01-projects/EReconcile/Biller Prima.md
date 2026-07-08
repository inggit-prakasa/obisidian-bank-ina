# Source
## IBMB Big Query
- translation_code in OB and O8


# Query Execute

```sql
-- 1. Create gmmappingreportDOP
insert into gmmappingreportdop ("fieldaccountdestination", "fieldaccountissuer", "fieldacquirerid", "fieldcardnumber", "fieldchargeamount", "fielddestination", "fieldissuerid", "fieldreciptnumber", "fieldstatus", "fieldterminalid", "fieldtransactiondate", "fieldtransactionfee", "fieldtransactionid", "fieldtransactiontime", "fieldtransactiontype", "parameterid", "sourceaccountdestination", "sourceaccountissuer", "sourceacquirerid", "sourcecardnumber", "sourcechargeamount", "sourcedestination", "sourceissuerid", "sourcereciptnumber", "sourcestatus", "sourceterminalid", "sourcetransactiondate", "sourcetransactionfee", "sourcetransactionid", "sourcetransactiontime", "sourcetransactiontype") 
  values ('source_account', 'debit_account_id', 'acquirer', 'card_no', 'fee_admin', 'recvbic', 'debtbic', 'partner_reference', 'status', 'product_name', 'settlement_date', 'fee_admin', 'journal_reference_id', 'transaction_date', 'transaction_type', gmparameter0_id, 9, 4, 2, 1, 9, 4, 4, 9, 4, 9, 9, 9, 9, 9, 1)

-- 2. Create gmparameter4
INSERT INTO "bina-erecon"."public"."gmparameter4" (lineno,idno,internaldata,externaldata,formulacolumn1,rules,formulacolumn2,tolerance,changeno,createby,createdate,changeby,changedate,flagamount) VALUES ('55','1','9','49','(total_amount)','=','(AMOUNT)','','0','sendhynugroho@gmail.com','2026-07-07 15:08:36.414723','sendhynugroho@gmail.com','2026-07-07 15:08:36.414723','1'),('55','2','9','49','(total_amount)','=','(AMOUNT)','','0','sendhynugroho@gmail.com','2026-07-07 15:08:36.414723','sendhynugroho@gmail.com','2026-07-07 15:08:36.414723','0'),('55','3','9','49','(partner_reference)','=','(REF-NO)','','0','sendhynugroho@gmail.com','2026-07-07 15:08:36.414723','sendhynugroho@gmail.com','2026-07-07 15:08:36.414723','0'),('55','4','9','49','(settlement_date)','=','(TANGGAL)','1','0','sendhynugroho@gmail.com','2026-07-07 15:08:36.414723','sendhynugroho@gmail.com','2026-07-07 15:08:36.414723','0');

-- 3. Insert gmfiletype1
INSERT INTO "bina-erecon"."public"."gmfiletype1" (lineno,idno,field,mappingtodatamart,startindex,endindex,row,changeno,createby,createdate,changeby,changedate,linenomappingdatamart) VALUES ('59','3','JAM','transaction_date','19','26','6','0','sendhynugroho@gmail.com','2026-07-07 15:05:57.196437','sendhynugroho@gmail.com','2026-07-07 15:05:57.196437','9'),('59','4','AMOUNT','base_amount','71','82','6','0','sendhynugroho@gmail.com','2026-07-07 15:05:57.196437','sendhynugroho@gmail.com','2026-07-07 15:05:57.196437','9'),('59','5','TRACE-NO','journal_reference_id','125','130','6','0','sendhynugroho@gmail.com','2026-07-07 15:05:57.196437','sendhynugroho@gmail.com','2026-07-07 15:05:57.196437','9'),('59','2','TANGGAL','settlement_date','9','16','6','0','sendhynugroho@gmail.com','2026-07-07 15:05:57.196437','sendhynugroho@gmail.com','2026-07-07 15:05:57.196437','9'),('59','1','REF-NO','partner_reference','21','33','7','0','sendhynugroho@gmail.com','2026-07-07 14:52:28.824281','sendhynugroho@gmail.com','2026-07-07 14:52:28.824281','9');

-- 4. Update filter 
UPDATE gmfilterreconciliation0 
SET ibmb = '0', biller = '1' 
WHERE linenoparameter = gmparameter0_id;

```