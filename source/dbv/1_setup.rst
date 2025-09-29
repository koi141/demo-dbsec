############################################
1. Database Vaultの有効化
############################################

Database Vaultの有効化、管理者ユーザーの作成、および各種構成作業を実施します。

.. topic:: 実施内容

    + Database Vault の構成前確認
    + Database Vault 管理者ユーザーの作成
    + Database Vault の構成
    + Database Vault の有効化
    + PDBでの構成と有効化


********************************
Database Vault 構成前の確認
********************************
Oracle Database Vault を有効化する前に、Database Vault および Oracle Label Security (以下OLS) の構築スクリプトを実行する必要があります。

.. admonition:: 注意

    Database Vaultは内部的にOLSのライブラリを使用しているため、構成時にOLSも自動で有効化されますが、OLSを別途使用しない限り、OLSのライセンスは不要です。

また、マルチテナントでの使用の場合、CDBでDatabase Vaultを構成および有効化した後で、PDBでも同様に有効化操作を行う必要があります。

CDB$ROOTに接続し、Database Vaultの構成および有効化状況を確認します。

.. code-block:: sql
    :caption: CDB

    SQL> SELECT * FROM CDB_DV_STATUS;

    NAME                STATUS             CON_ID
    ------------------- -------------- ----------
    DV_CONFIGURE_STATUS FALSE                   1
    DV_ENABLE_STATUS    FALSE                   1
    DV_APP_PROTECTION   NOT CONFIGURED          1
    DV_CONFIGURE_STATUS FALSE                   3
    DV_ENABLE_STATUS    FALSE                   3
    DV_APP_PROTECTION   NOT CONFIGURED          3

    6 rows selected.

|

*********************************************************************************
【CDB$ROOT】Database Vault管理者ユーザーの作成（C##DVOWNER、C##DVACCTMGR）
*********************************************************************************

| Database Vaultを有効化する際、管理者ユーザーとして Database Vault所有者 (DV_OWNERロール) と アカウント管理者 (DV_ACCTMGRロール) の2つのアカウントを指定する必要があります。
| ここでは以下のユーザーを用いるため、2つのユーザーをそれぞれ作成します。今後の手順でこの二人のユーザーはよく使いますので、各セッションを別端末などで用意しておくと今後の作業がスムーズに進めやすいかと思います。

+ **C##DVOWNER** ユーザー: Database Vaultのロールや構成を管理します。
+ **C##DVACCTMGR** ユーザー: データベースのユーザーアカウントを管理します。



| CDBルートで DV_OWNER ロールと DV_ACCTMGR ロールを付与された共通ユーザーは、PDBでも同じロールを持つことができるように構成できます。
| DV_ACCTMGR ロールは CDBルートの共通ユーザーに対して共通ロールとして付与されます。
一方、DV_OWNER ロールは、Database Vault の構成時にローカルユーザーに付与することも、CDB全体で有効な共通ユーザーに共通ロールとして付与することもできます。
| ただし、DV_OWNER を共通ユーザーにローカルロールとして付与した場合、そのユーザーは他の PDB では DV_OWNER ロールを使用できなくなるため注意が必要です。

今回の構成では、CDBルート・PDB いずれにおいても、共通ユーザー（上記２つのユーザー）を DV 管理者として使用します。
このため、PDBでの有効化時にも共通ユーザーを再利用します。

共通ユーザーを作成する前に、共通ユーザーの接頭辞（通常は ``C##`` ）が正しいかどうか確認しておきます。

.. code-block:: sql
    :caption: CDB

    SQL> set markup csv on
    SQL> select name, value from v$parameter where name = 'common_user_prefix';
    "NAME"              ,"VALUE"
    "common_user_prefix","C##"


以下を実行してユーザーをそれぞれ作成します。


.. code-block:: sql
    :caption: CDB

    grant create session, set container to C##DVOWNER identified by Welcome1#Welcome1# container = all;
    grant create session, set container to C##DVACCTMGR identified by Welcome1#Welcome1# container = all;

|

************************************
【CDB$ROOT】Database Vaultの構成
************************************
作成した2つのユーザーを指定して、Database Vaultを構成します。


.. code-block:: sql
    :caption: CDB

    BEGIN
        CONFIGURE_DV (
            dvowner_uname        => 'C##DVOWNER',    
            dvacctmgr_uname      => 'C##DVACCTMGR',
            force_local_dvowner  => FALSE);
    END;
    /

``force_local_dvowner`` をFALSEに設定すると、共通ユーザーはこのCDBルートに関連付けられているPDBの DV_OWNER 権限を持つことができます。
TRUEに設定すると、共通 DV_OWNER ユーザーはCDBルートにのみ DV_OWNER ロール権限を持つように制限されます。

CDBの構成ステータスが TRUE になっていることを確認します。

.. code-block:: sql
    :caption: CDB

    SQL> SELECT * FROM CDB_DV_STATUS;
    "NAME"               ,"STATUS"        ,"CON_ID"
    "DV_CONFIGURE_STATUS","TRUE"          ,1
    "DV_ENABLE_STATUS"   ,"FALSE"         ,1
    "DV_APP_PROTECTION"  ,"NOT CONFIGURED",1
    "DV_CONFIGURE_STATUS","FALSE"         ,3
    "DV_ENABLE_STATUS"   ,"FALSE"         ,3
    "DV_APP_PROTECTION"  ,"NOT CONFIGURED",3

    6 rows selected.

utlrp.sqlスクリプトを実行し、無効化状態となっているオブジェクトをコンパイルします。

.. code-block:: sql
    :caption: CDB

    SQL> @?/rdbms/admin/utlrp.sql


問題なく、実行が完了することを確認します。

|

*******************************************
【CDB$ROOT】Database Vaultを有効化する
*******************************************
Database Vaultをマルチテナント環境(CDB)で有効化する際には、大きく分けて「通常(非厳密)モード」と「厳密モード」の2つの動作モードを選択できます。
これらのモードは、CDB全体でDatabase Vaultが有効化されている際に、PDBごとの Database Vaultの有効化状態がどのように扱われるかを制御します。

+ 通常モード
    CDBでDatabase Vaultが有効化されている場合でも、PDB単位で Database Vault が有効化されているかどうかにかかわらず、そのPDBは通常通り機能を継続します。
    つまり、CDBでは Database Vault が有効であっても、PDBレベルで無効な状態のままでもPDBは使い続けることができます。

+ 厳密モード
    厳密モードでは、CDBがDatabase Vault有効状態になった時点で、PDBを読み書きモードでオープンするにはそのPDBでもDatabase Vaultが構成・有効化されている必要があります。
    簡単に言えば、「CDBでDatabase Vaultが有効なら、すべてのPDBもDatabase Vaultを有効化していないと開けない」という制限が課されます。

今回はPDBだけで Database Vault を使用していきますので、「通常モード」で有効化していきます。

先ほど作成し、DV管理者として指定した ``C##DVOWNER`` ユーザーでCDBに接続します。

.. code-block:: sql
    :caption: CDB

    $ sqlplus C##DVOWNER/<password>

    SQL> show user con_name
    USER is "C##DVOWNER"

    CON_NAME
    ------------------------------
    CDB$ROOT


通常モードで有効化します。

.. code-block:: sql
    :caption: CDB

    SQL> EXEC DBMS_MACADM.ENABLE_DV;


CDB_DV_STATUSを確認し、有効化されていることを確認します。

.. code-block:: sql
    :caption: CDB

    SQL> SELECT * FROM CDB_DV_STATUS;
    "NAME"               ,"STATUS"        ,"CON_ID"
    "DV_CONFIGURE_STATUS","TRUE"          ,1
    "DV_ENABLE_STATUS"   ,"TRUE"          ,1
    "DV_APP_PROTECTION"  ,"NOT CONFIGURED",1


SYSユーザーで再び接続し、Database Vaultの設定を完全に反映させるためCDBを再起動します。


.. code-block:: sql
    :caption: CDB

    SQL> shutdown immediate
    SQL> startup

    -- PDBがオープンされていることを確認
    SQL> show pdbs;

        CON_ID CON_NAME                       OPEN MODE  RESTRICTED
    ---------- ------------------------------ ---------- ----------
            2 PDB$SEED                       READ ONLY  NO
            3 FREEPDB1                       READ WRITE NO

    -- 再起動後、Database VaultとOLSが有効化されていることを確認

    SQL> set markup csv on
    SQL> SELECT parameter, VALUE FROM V$OPTION WHERE PARAMETER IN ('Oracle Database Vault','Oracle Label Security');
    "PARAMETER"            ,"VALUE"
    "Oracle Label Security","TRUE"
    "Oracle Database Vault","TRUE"

|

**************************************
【PDB】Database Vaultの構成と有効化
**************************************
FREEPDB1に接続し、改めてPDBのオプションの状況を確認します。

.. code-block:: sql
    :caption: PDB

    SQL> set markup csv on
    SQL> SELECT parameter, VALUE FROM V$OPTION WHERE PARAMETER IN ('Oracle Database Vault','Oracle Label Security');
    "PARAMETER"            ,"VALUE"
    "Oracle Label Security","FALSE"
    "Oracle Database Vault","FALSE"

    -- PDBのdatabase Vaultのステータスを確認します。
    SQL> SELECT * FROM DBA_DV_STATUS;
    "NAME"               ,"STATUS"
    "DV_CONFIGURE_STATUS","FALSE"
    "DV_ENABLE_STATUS"   ,"FALSE"
    "DV_APP_PROTECTION"  ,"NOT CONFIGURED"



FREEPDB1 にSYSユーザーで接続し、Database Vaultを構成します。

.. code-block:: sql
    :caption: PDB

    BEGIN
        CONFIGURE_DV (
            dvowner_uname        => 'C##DVOWNER',    
            dvacctmgr_uname      => 'C##DVACCTMGR');
    END;
    /

.. code-block:: sql
    :caption: PDB

    -- 構成ステータスがTRUEになっていることを確認
    SQL> SELECT * FROM DBA_DV_STATUS;
    "NAME"               ,"STATUS"
    "DV_CONFIGURE_STATUS","TRUE"
    "DV_ENABLE_STATUS"   ,"FALSE"
    "DV_APP_PROTECTION"  ,"NOT CONFIGURED"



``utlrp.sql`` スクリプトを実行し、無効化状態となっているオブジェクトをコンパイルします。

.. code-block:: sql
    :caption: PDB

    SQL> @?/rdbms/admin/utlrp.sql

問題なく、実行が完了することを確認します。


先ほど構成した Vault所有者ユーザーとして、PDBに接続します。

.. code-block:: sql
    :caption: PDB

    $ sqlplus c##dvowner/<password>@localhost:1521/FREEPDB1

    SQL> show user con_name
    USER is "C##DVOWNER"

    CON_NAME
    ------------------------------
    FREEPDB1

    -- Database Vaultを有効化

    SQL> EXEC DBMS_MACADM.ENABLE_DV;


CDBにSYSユーザーで接続し、PDBを再起動します。

.. code-block:: sql
    :caption: CDB

    SQL> conn / as sysdba
    Connected.
    SQL> show user con_name
    USER is "SYS"

    CON_NAME
    ------------------------------
    CDB$ROOT

    SQL> alter pluggable database freepdb1 close immediate;

    SQL> alter pluggable database freepdb1 open;

Database VaultおよびOLSが有効化されたことを確認します

.. code-block:: sql
    :caption: PDB

    SQL> SELECT * FROM DBA_DV_STATUS;

    NAME                STATUS
    ------------------- --------------
    DV_CONFIGURE_STATUS TRUE
    DV_ENABLE_STATUS    TRUE
    DV_APP_PROTECTION   NOT CONFIGURED

    SQL> col description for a40
    SQL> SELECT * FROM DBA_OLS_STATUS;

    NAME                 STATU DESCRIPTION
    -------------------- ----- ----------------------------------------
    OLS_CONFIGURE_STATUS TRUE  Determines if OLS is configured
    OLS_ENABLE_STATUS    TRUE  Determines if OLS is enabled

Database Vault の事前設定はこちらで完了です。
次の手順では、レルムの作成やオブジェクトに対する認可を行っていきます。