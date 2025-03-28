###########################
1. TDEの事前準備
###########################

.. topic:: 実施内容

   + TDEに関連する初期化パラメータ設定する
   + キーストアおよびマスター暗号鍵を作成する


****************************
初期化パラメータの確認
****************************

TDEに関連する初期化パラメータは2つあります。

+ `wallet_root <https://docs.oracle.com/cd/F82042_01/refrn/WALLET_ROOT.html>`__ :  暗号化キー（TDEキー）や証明書などを保存する場所を指定
+ `tde_configuration <https://docs.oracle.com/cd/F82042_01/refrn/TDE_CONFIGURATION.html>`__ : TDEの設定を定義

それぞれを確認していきましょう。


wallet_root
============================

FREEPDB1に接続し、現在の値を確認します。


.. code:: sql

   -- 結果をcsv形式で出力
   SQL> set markup csv on

   -- wallet_rootの値を確認
   SQL> select inst_id, name, value, issys_modifiable from gv$parameter where name = 'wallet_root';
   "INST_ID","NAME"       ,"VALUE","ISSYS_MODIFIABLE"
   1        ,"wallet_root",       ,"FALSE"

デフォルトでは何も設定されていないことが確認できます。  
``ISSYS_MODIFIABLE`` 列の値が ``FALSE`` であるため、設定の反映に再起動が必要になります。（つまり、静的パラメータであることを意味します。）



tde_configuration
============================

同様に、現在の値を確認します。

.. code:: sql

   SQL> select inst_id, name, value, issys_modifiable from gv$parameter where name = 'tde_configuration';
   "INST_ID","NAME"             ,"VALUE","ISSYS_MODIFIABLE"
   1        ,"tde_configuration",       ,"IMMEDIATE"

こちらもデフォルト値はなく、何も設定されていないことが確認できます。  
``ISSYS_MODIFIABLE`` は ``IMMEDIATE`` のため、この設定は即時反映される動的パラメータであることが分かります。



****************************
初期化パラメータの設定
****************************

では、各パラメータを設定していきます。

wallet_root
============================

TDEの暗号化キーはOracle Walletに保存されます。  
``wallet_root`` でこのWalletを格納するディレクトリのパスを指定します。

ディレクトリは任意の場所に指定できますが、ここでは ``/opt/oracle/admin/FREE/wallet`` を指定します。
この wallet ディレクトリは作成されていませんので、事前に作成しておきます。

.. code-block:: bash

   $ mkdir -p /opt/oracle/admin/FREE/wallet


次にCDBに接続し、以下のコマンドでパラメータを設定します。 
PDBから実行はできませんので、CDBから設定を行ってください。

.. code-block:: sql

   SQL> alter system set wallet_root='/opt/oracle/admin/FREE/wallet' scope=spfile;

   -- 静的パラメータのため、再起動するまでは反映されない
   SQL> select inst_id, name, value, issys_modifiable from gv$parameter where name = 'wallet_root';
   "INST_ID","NAME"       ,"VALUE","ISSYS_MODIFIABLE"
   1        ,"wallet_root",       ,"FALSE"

   -- 再起動
   SQL> shutdown immediate
   SQL> startup

   -- 再起動後、設定が反映されている
   SQL> select inst_id, name, value, issys_modifiable from gv$parameter where name = 'wallet_root';
   "INST_ID","NAME"       ,"VALUE"                        ,"ISSYS_MODIFIABLE"
   1        ,"wallet_root","/opt/oracle/admin/FREE/wallet","FALSE"


tde_configuration
============================

``tde_configuration`` はTDEで使用されるキーストアの種類を設定します。  
19cからは分離モードがサポートされ、PDBごとに固有のキーストアを使用できるようになりました。


サポートされるキーストアは以下の通りです。

.. image:: ./images/サポートされるキーストア.png


詳細は `こちら <https://docs.oracle.com/cd/F82042_01/asoag/introduction-to-transparent-data-encryption.html>`__ でご確認ください。



有効化すると設定した値によって ``wallet_root`` 配下に以下のディレクトリが作成されます。そのため、このパラメータ設定のためには ``wallet_root`` を先に有効にしておく必要があります。

:FILE: ``<WALLET_ROOT>/tde``
:Oracle Key Vault: ``<WALLET_ROOT>/okv``

今回はデモですので、DBサーバーにキーストアを設置することとします。

以下のコマンドで ``tde_configuration`` を設定します。

.. code:: sql

   SQL> alter system set tde_configuration='keystore_configuration=file' scope=both;

   -- すぐに反映されている
   SQL> select inst_id, name , value , issys_modifiable from gv$parameter where name = 'tde_configuration';
   "INST_ID","NAME"             ,"VALUE"                      ,"ISSYS_MODIFIABLE"
   1        ,"tde_configuration","keystore_configuration=file","IMMEDIATE"

CDBで設定を行った場合、PDBはCDBからその値を継承します。



****************************
キーストアの作成
****************************

| 暗号化鍵を格納するためのキーストアを作成します。キーストアのマスター鍵管理はSYSKM権限以上が必要になります。
| キーストア操作の専用ユーザーとしてsyskmユーザーも用意されていますが、SYSユーザーでも可能です。どちらかを使用してください。

以下のコマンドでキーストアを作成します。デフォルトではPKCS#12ベースのキーストレージファイルに保存されます。（参考:  `ADMINISTER KEY MANAGEMENT <https://docs.oracle.com/en/database/oracle/oracle-database/23/sqlrf/ADMINISTER-KEY-MANAGEMENT.html>`__ ）

.. code-block:: sql
   :caption: CDBで実行 (SYSユーザー or SYSKMユーザー)

   SQL> administer key management create keystore identified by OracleKM123#;


このコマンドを実行すると ``<wallet_root>/tde`` ディレクトリが作成され、その中に ``ewallet.p12`` が作成されます。

.. code:: bash

   $ pwd && tree
   /opt/oracle/admin/FREE/wallet
   .
   └── tde
      └── ewallet.p12


キーストアが正しく作成されたことは、 ``V$ENCRYPTION_WALLETビュー`` からも確認することができます。

.. code-block:: sql
   :caption: CDBで実行 (syskmユーザー)

   SQL> select * from v$encryption_wallet;
   "WRL_TYPE","WRL_PARAMETER"                     ,"STATUS","WALLET_TYPE","WALLET_ORDER","KEYSTORE_MODE","FULLY_BACKED_UP","CON_ID"
   "FILE"    ,"/opt/oracle/admin/FREE/wallet/tde/","CLOSED","UNKNOWN"    ,"SINGLE"      ,"NONE"         ,"UNDEFINED"      ,1


| こちらの結果からわかる通り、キーストアの状態が `CLOSED`` となっています。
| この状態ではキーストアは使用できませんので、次のコマンドにてキーストアをOPENにします。

.. code-block:: sql

   SQL> administer key management set keystore open identified by OracleKM123#;
   
   -- STATUS列がOPENになったことを確認
   SQL> select * from v$encryption_wallet;
   "WRL_TYPE","WRL_PARAMETER"                     ,"STATUS"            ,"WALLET_TYPE","WALLET_ORDER","KEYSTORE_MODE","FULLY_BACKED_UP","CON_ID"
   "FILE"    ,"/opt/oracle/admin/FREE/wallet/tde/","OPEN_NO_MASTER_KEY","PASSWORD"   ,"SINGLE"      ,"NONE"         ,"UNDEFINED"      ,1

このように、STATUS列が `OPEN_NO_MASTER_KEY` に変わり、キーストアが正常に開かれたことが確認できます。  
これで、キーストアを使用して暗号化操作を行う準備が整いました。


****************************
マスター暗号鍵の作成
****************************

続いてマスター暗号鍵を作成します。今回はCDB、PDBを一括で暗号化するために統合モードで鍵を作成します。

.. code-block:: sql
   :caption: CDBで実行 (SYSユーザー or SYSKMユーザー)

   SQL> administer key management set key using tag 'v1.0_MEK_AllContainer' identified by OracleKM123# with backup container = ALL;


| ``using tag`` 句は省略可能ですが、管理のために付けておくことをお勧めします。  
| 次に、PDBでウォレットとマスター暗号鍵が正しく認識されているかを確認します。

.. code-block:: sql
   :caption: FREEPDB1で実行 (SYSユーザー)

   -- PDBでウォレットの状態を確認
   SQL> select * from v$encryption_wallet;
   "WRL_TYPE","WRL_PARAMETER","STATUS","WALLET_TYPE","WALLET_ORDER","KEYSTORE_MODE","FULLY_BACKED_UP","CON_ID"
   "FILE"    ,               ,"OPEN"  ,"PASSWORD"   ,"SINGLE"      ,"UNITED"       ,"NO"             ,3

   -- PDBでマスター暗号鍵を認識しているか確認
   SQL> select key_id, tag, creator, user, key_use, keystore_type, activating_dbname from v$encryption_keys;
   "KEY_ID"      ,"TAG"                  , "CREATOR","USER","KEY_USE"   ,"KEYSTORE_TYPE"    ,"ACTIVATING_DBNAME"
   "AU1kv...AAAA","v1.0_MEK_AllContainer", "SYSKM"  ,"SYS" ,"TDE IN PDB","SOFTWARE KEYSTORE","FREE"


**参考**

+ `V$ENCRYPTION_WALLET <https://docs.oracle.com/en/database/oracle/oracle-database/23/refrn/V-ENCRYPTION_WALLET.html>`__ : ウォレットの状態とTDEウォレットの場所に関する情報を表示  
+ `V$ENCRYPTION_KEYS <https://docs.oracle.com/en/database/oracle/oracle-database/23/refrn/V-ENCRYPTION_KEYS.html>`__ : マスターキーの説明属性を表示


これで準備が整いましたので、次の手順から実際に表領域の暗号化を行っていきます。