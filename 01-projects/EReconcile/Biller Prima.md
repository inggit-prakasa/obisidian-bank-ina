# Source
## IBMB Big Query
- translation_code in OB and O8


# Query Execute

```sql
-- 0. Select gmparameter0 and gmfiletype0 with partipant is Prima and channel biller
select * from gmparameter0 where channelname = 'Biller' and thirdpartyname = 'Prima'; -- 55
select * from gmfiletype0 where channel = 'Biller' and participant = 'Prima'; -- 59

-- 1. Create gmmappingreportDOP
INSERT INTO gmmappingreportdop (parameterid,sourcetransactionid,fieldtransactionid,sourcetransactiondate,fieldtransactiondate,sourcetransactiontime,fieldtransactiontime,sourceacquirerid,fieldacquirerid,sourceissuerid,fieldissuerid,sourceterminalid,fieldterminalid,sourcedestination,fielddestination,sourcereciptnumber,fieldreciptnumber,sourcetransactionfee,fieldtransactionfee,sourceaccountissuer,fieldaccountissuer,sourceaccountdestination,fieldaccountdestination,sourcetransactiontype,fieldtransactiontype,sourcechargeamount,fieldchargeamount,sourcecardnumber,fieldcardnumber,sourcestatus,fieldstatus) VALUES (gmparameter0_id,9,'journal_reference_id',9,'settlement_date',9,'transaction_date',2,'acquirer',4,'debtbic',9,'product_name',4,'recvbic',9,'partner_reference',9,'fee_admin',9,'source_account',2,'dest_acc_number',1,'transaction_type',9,'fee_admin',1,'card_no',4,'status');

-- 2. Create gmparameter4
DELETE from gmparameter4 where lineno = gmparameter0_id;

INSERT INTO gmparameter4 (lineno,idno,internaldata,externaldata,formulacolumn1,rules,formulacolumn2,tolerance,changeno,createby,createdate,changeby,changedate,flagamount) VALUES 
(gmparameter0_id,1,9,gmfiletype0_id,'(total_amount)','=','(AMOUNT)','',0,'System',NOW(),'System',NOW(),1),
(gmparameter0_id,2,9,gmfiletype0_id,'(partner_reference)','=','(REF-NO)','',0,'System',NOW(),'System',NOW(),0),
(gmparameter0_id,3,9,gmfiletype0_id,'(settlement_date)','=','(TANGGAL)','1',0,'System',NOW(),'System',NOW(),0);

-- 3. Insert gmfiletype1
DELETE from gmfiletype1 where lineno = gmfiletype0_id;

INSERT INTO gmfiletype1 (lineno,idno,field,mappingtodatamart,startindex,endindex,row,changeno,createby,createdate,changeby,changedate,linenomappingdatamart) VALUES 
(gmfiletype0_id,1,'REF-NO','partner_reference',21,33,7,0,'System',NOW(),'System',NOW(),9),
(gmfiletype0_id,2,'TANGGAL','settlement_date',9,17,6,0,'System',NOW(),'System',NOW(),9),
(gmfiletype0_id,3,'JAM','transaction_date',19,27,6,0,'System',NOW(),'System',NOW(),9),
(gmfiletype0_id,4,'AMOUNT','base_amount',71,82,6,0,'System',NOW(),'System',NOW(),9),
(gmfiletype0_id,5,'TRACE-NO','journal_reference_id',125,130,6,0,'System',NOW(),'System',NOW(),9);

-- 4. Update filter 
UPDATE gmfilterreconciliation0 
SET ibmb = '0', biller = '1' 
WHERE linenoparameter = gmparameter0_id;
```