
- Memastikan transaksi type ACMD di dwh itu recipt numbernya berbeda dari transaction type ACMD yang lain
	- Jika memang kejadian ini adalah kejadian yang biasa terjadi maka kita akan merubah ETL untuk ACMD menggunakan kolom payment_details yang menjadi end_to_end_id
	- Jika kejadian ini merupakan kejanggalan transaksi maka di tim erecon tidak merubah apa-apa dan menjadi issue IBB atau interbank-transfer saja

- Biller otto digital akan dilakukan test sampai tanggla 2 juli
- BiFast Enhancement
	- Akan enhance transaksi konven medialion / transaction type ACND
	- Akan enhance untuk 