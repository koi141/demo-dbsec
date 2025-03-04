############################################
1. Database Vaultの有効化
############################################

Database Vault (DB Vault) の有効化、管理者ユーザーの作成、および各種構成作業を実施します。

**実施内容**

+ Database Vault の構成前確認
+ Database Vault 管理者ユーザーの作成
+ Database Vault の構成
+ Database Vault の有効化
+ PDBでの構成と有効化


********************************
Database Vault の構成前確認
********************************
Oracle Database Vaultを有効化する前に、Database VaultおよびOracle Label Security(OLS)のための構築スクリプトを実行する必要があります。
DB Vault構成時にOLSが有効化されます。別にOLSを使用する場合を除いて、OLSのライセンスは必要ありません。

また、マルチテナントでの使用の場合、CDBでDB Vaultを構成および有効化した後で、PDBでも同様の操作を行う必要があります。

CDBルートでDV_OWNERロールとDV_ACCTMGRロールを割り当てられた共通ユーザーも、PDBで同じロールを持つことができます。
PDBでは、同じ共通ユーザーを使用してDatabase Vaultを構成および有効化することも、別のPDBローカル・ユーザーを使用することもできます。DV_ACCTMGRロールは、CDBルートの共通ユーザーに共通に付与されます。
Database VaultをCDBルートに構成および有効化するときに、DV_OWNERをローカルに、またはCDBルート共通ユーザーに共通に付与できます。
DV_OWNERを共通ユーザーにローカルに付与すると、共通DV_OWNERユーザーは、どのPDBでもこのロールを使用できなくなります。

今回はDV管理者としては共通ユーザを使用することとします。

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

*************************************************
(CDB$ROOT) Database Vault管理者ユーザーの作成
*************************************************

Database Vaultを有効化する際、DB Vault所有者 (DV_OWNERロール) と アカウント管理者 (DV_ACCTMGRロール) の2つのアカウントを指定する必要があります。
ここでは以下のユーザーを用いるため、2つのユーザーをそれぞれ作成します。

+ **C##DVOWNER**: Database Vaultのロールや構成を管理します。
+ **C##DVACCTMGR**: データベースのユーザーアカウントを管理します。

PDBで有効化する際も管理者アカウントを指定する必要がありますが、CDBの共通ユーザを指定することも可能のため、ここでは共通ユーザーをPDBでも繰り返し使用します。

念のため、共通ユーザの接頭辞を確認します。

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

********************************
(CDB$ROOT) Database Vaultの構成
********************************
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

force_local_dvownerをFALSEに設定すると、共通ユーザーは、このCDBルートに関連付けられているPDBのDV_OWNER権限を持つことができます。
TRUEに設定すると、共通DV_OWNERユーザーはCDBルートにのみDV_OWNERロール権限を持つように制限されます。

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


***************************************
(CDB$ROOT) Database Vaultを有効化する
***************************************
DB Vaultをマルチテナント環境(CDB)で有効化する際には、大きく分けて「通常(非厳密)モード」と「厳密モード」の2つの動作モードを選択できます。
これらのモードは、CDB全体でDB Vaultが有効化されている際に、PDB(Pluggable Database)ごとのDB Vault有効化状態がどのように扱われるかを制御します。

+ 通常モード
    CDBでDB Vaultが有効化されている場合でも、PDB単位でDB Vaultが有効化されているかどうかにかかわらず、そのPDBは通常通り機能を継続します。
    つまり、CDBではDB Vaultが有効であっても、PDBレベルで無効な状態のままでもPDBは使い続けることができます。

+ 厳密モード
    厳密モードでは、CDBがDB Vault有効状態になった時点で、PDBを読み書きモードでオープンするにはそのPDBでもDB Vaultが構成・有効化されている必要があります。
    簡単に言えば、「CDBでDB Vaultが有効なら、すべてのPDBもDB Vaultを有効化していないと開けない」という制限が課されます。

今回はPDBだけDVを有効化していきますので、「通常モード」で有効化していきます。

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


SYSユーザーで再び接続し、DB Vaultの設定を完全に反映させるためCDBを再起動します。


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

    -- 再起動後、DVとOLSが有効化されていることを確認

    SQL> set markup csv on
    SQL> SELECT parameter, VALUE FROM V$OPTION WHERE PARAMETER IN ('Oracle Database Vault','Oracle Label Security');
    "PARAMETER"            ,"VALUE"
    "Oracle Label Security","TRUE"
    "Oracle Database Vault","TRUE"


**************************************
(PDB) Database Vaultの構成と有効化
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



FREEPDB1にSYSユーザーで接続し、DB Vaultを構成します。

.. code-block:: sql
    :caption: PDB

    BEGIN
        CONFIGURE_DV (
            dvowner_uname        => 'C##DVOWNER',    
            dvacctmgr_uname      => 'C##DVACCTMGR');
    END;
    /

    -- 構成ステータスがTRUEになっていることを確認
    SQL> SELECT * FROM DBA_DV_STATUS;
    "NAME"               ,"STATUS"
    "DV_CONFIGURE_STATUS","TRUE"
    "DV_ENABLE_STATUS"   ,"FALSE"
    "DV_APP_PROTECTION"  ,"NOT CONFIGURED"



utlrp.sqlスクリプトを実行し、無効化状態となっているオブジェクトをコンパイルします。

.. code-block:: sql
    :caption: PDB

    SQL> @?/rdbms/admin/utlrp.sql

問題なく、実行が完了することを確認します。


先ほど構成したDB Vault所有者ユーザーとして、PDBに接続します。

.. code-block:: sql
    :caption: PDB

    $ sqlplus c##dvowner/<password>@localhost:1521/FREEPDB1

    SQL> show user con_name
    USER is "C##DVOWNER"

    CON_NAME
    ------------------------------
    FREEPDB1

    -- DB Vaultを有効化

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

DB VaultおよびOLSが有効化されたことを確認します

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

