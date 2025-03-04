##############################################
3. キーストアを自動でオープンするようにする
##############################################

Databaseを再起動した際、以下の結果のようになりKeystoreは再びクローズします。

.. code-block:: sql
    :caption: CDBにて実行

    SQL> select * from v$encryption_wallet;
    "WRL_TYPE","WRL_PARAMETER","STATUS","WALLET_TYPE","WALLET_ORDER","KEYSTORE_MODE","FULLY_BACKED_UP","CON_ID"
    "FILE","/opt/oracle/admin/FREE/wallet/tde/","CLOSED","UNKNOWN","SINGLE","NONE","UNDEFINED",1
    "FILE",,"CLOSED","UNKNOWN","SINGLE","UNITED","UNDEFINED",2
    "FILE",,"CLOSED","UNKNOWN","SINGLE","UNITED","UNDEFINED",3


そのため、以下のコマンドにて再び明示的にオープンにする必要がありますが、この操作が少々面倒な場面もあります。

.. code-block:: sql

    administer key management set keystore open identified by <password>;


そこで、ここでは自動ログインキーストアを作成し、Databaseが起動した際に自動でオープンするように設定してみます。


*********************************************
1. キーストアをオープンする
*********************************************

キーストアがオープンしていない場合、こちらの手順にてキーストアをオープンします。

.. code-block:: sql
    :caption: CDBにて実行

    SQL> select * from v$encryption_wallet;
    "WRL_TYPE","WRL_PARAMETER","STATUS","WALLET_TYPE","WALLET_ORDER","KEYSTORE_MODE","FULLY_BACKED_UP","CON_ID"
    "FILE","/opt/oracle/admin/FREE/wallet/tde/","CLOSED","UNKNOWN","SINGLE","NONE","UNDEFINED",1
    "FILE",,"CLOSED","UNKNOWN","SINGLE","UNITED","UNDEFINED",2
    "FILE",,"CLOSED","UNKNOWN","SINGLE","UNITED","UNDEFINED",3

    SQL> administer key management set keystore open identified by OracleKM123# container = all;

    keystore altered.

    -- オープンしていることを確認する
    SQL> select * from v$encryption_wallet;
    "WRL_TYPE","WRL_PARAMETER","STATUS","WALLET_TYPE","WALLET_ORDER","KEYSTORE_MODE","FULLY_BACKED_UP","CON_ID"
    "FILE","/opt/oracle/admin/FREE/wallet/tde/","OPEN","PASSWORD","SINGLE","NONE","NO",1
    "FILE",,"OPEN","PASSWORD","SINGLE","UNITED","NO",2
    "FILE",,"OPEN","PASSWORD","SINGLE","UNITED","NO",3

``container = all`` 句を忘れた場合、CDBだけがオープンしますので、その後PDBに別途接続してオープンするコマンドを実行すれば問題ありません。


*********************************************
2. 自動ログインのキーストアを作成
*********************************************

キーストアをオープンするために自動でログインするキーストアを作成します。

.. code-block:: sql
    :caption: CDBにて実行

    SQL> administer key management create auto_login keystore from keystore identified by OracleKM123#;

    keystore altered.

    SQL> select * from v$encryption_wallet;
    "WRL_TYPE","WRL_PARAMETER","STATUS","WALLET_TYPE","WALLET_ORDER","KEYSTORE_MODE","FULLY_BACKED_UP","CON_ID"
    "FILE","/opt/oracle/admin/FREE/wallet/tde/","OPEN","PASSWORD","SINGLE","NONE","NO",1
    "FILE",,"OPEN","PASSWORD","SINGLE","UNITED","NO",2
    "FILE",,"OPEN","PASSWORD","SINGLE","UNITED","NO",3
    
    -- 作成されたキーストアを確認
    SQL> !ls -l /opt/oracle/admin/FREE/wallet/tde/
    total 24
    -rw-------. 1 oracle oinstall 7016 Feb 21 11:07 cwallet.sso
    -rw-------. 1 oracle oinstall 2555 Nov 25 06:27 ewallet_2024112506271655.p12
    -rw-------. 1 oracle oinstall 4059 Nov 25 08:10 ewallet_2024112508100476.p12
    -rw-------. 1 oracle oinstall 6955 Nov 25 08:10 ewallet.p12


この時点で ``cwallet.sso`` が作成され、 ``ewallet`` のバックアップも作成されたことが確認できると思います。
CDBで実行していますので、上記ではCDBの鍵のみが登録されていますが、今回の環境ではPDBも同じ鍵を共有しているためPDBも自動でオープンされることになります。

一旦CDBから以下の操作を行いDBを再起動します。

.. code-block:: sql
    :caption: CDBにて実行

    -- DBをシャットダウン
    SQL> shu immediate
    
    -- DBを立ち上げる
    SQL> startup

    SQL> select * from v$encryption_wallet;
    "WRL_TYPE","WRL_PARAMETER","STATUS","WALLET_TYPE","WALLET_ORDER","KEYSTORE_MODE","FULLY_BACKED_UP","CON_ID"
    "FILE","/opt/oracle/admin/FREE/wallet/tde/","OPEN","AUTOLOGIN","SINGLE","NONE","NO",1
    "FILE",,"OPEN","AUTOLOGIN","SINGLE","UNITED","NO",2
    "FILE",,"OPEN","AUTOLOGIN","SINGLE","UNITED","NO",3

改めて確認すると、STATUS列が ``OPEN`` 、WALLET_TYPE列が ``AUTOLOGIN`` となっており、再起動を行っても自動でオープンされたことがわかります。


以上でTDEのデモは終了です。