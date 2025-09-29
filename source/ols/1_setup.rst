############################################
1. Oracle Label Securityの準備
############################################
この手順では、Oracle Label Security (OLS) を有効化し、ポリシーを作成・設定します。

.. topic:: 実施内容
    
    + 表の準備
    + OLSの構成確認
    + OLSの有効化
    + ポリシーの作成と有効化


****************************
表の準備
****************************
JOB_HISTORY表をコピーし、ラベルを設定する表を ``HR.JOB_HISTORY_4OLS`` として別途用意します。

.. code-block:: sql
    :caption: FREEPDB1

    SQL> CREATE TABLE hr.job_history_4ols AS SELECT * FROM hr.job_history WHERE 1=0;

    SQL> INSERT INTO hr.job_history_4ols SELECT * FROM hr.job_history;



作成した表を確認します

.. code-block:: sql
    :caption: FREEPDB1

    SQL> select * from hr.job_history_4ols;

    EMPLOYEE_ID START_DAT END_DATE  JOB_ID     DEPARTMENT_ID
    ----------- --------- --------- ---------- -------------
            102 13-JAN-11 24-JUL-16 IT_PROG               60
            101 21-SEP-07 27-OCT-11 AC_ACCOUNT           110
            101 28-OCT-11 15-MAR-15 AC_MGR               110
            201 17-FEB-14 19-DEC-17 MK_REP                20
            114 24-MAR-16 31-DEC-17 ST_CLERK              50
            122 01-JAN-17 31-DEC-17 ST_CLERK              50
            200 17-SEP-05 17-JUN-11 AD_ASST               90
            176 24-MAR-16 31-DEC-16 SA_REP                80
            176 01-JAN-17 31-DEC-17 SA_MAN                80
            200 01-JUL-12 31-DEC-16 AC_ACCOUNT            90

    10 rows selected.



****************************
OLSの構成確認
****************************

現在、OLSが構成されているかを確認します。DBA_SA_USER_PRIVSデータ・ディクショナリ・ビューを使用

.. code-block:: sql
    :caption: FREEPDB1

    SQL> col status for a20
    SQL> col DESCRIPTION for a50
    SQL> set lines 200
    SQL> SELECT * FROM DBA_OLS_STATUS;

    NAME                 STATUS               DESCRIPTION
    -------------------- -------------------- --------------------------------------------------
    OLS_CONFIGURE_STATUS FALSE                Determines if OLS is configured
    OLS_ENABLE_STATUS    FALSE                Determines if OLS is enabled


ステータスの意味はそれぞれ以下になります。

+ OLS_CONFIGURE_STATUS: Oracle Label Securityが構成されているかどうかを判断します。
+ OLS_ENABLE_STATUS: Oracle Label Securityが有効化されているかどうかを判断します。


CDB_OLS_STATUSビューを使用することで、CDB全体のOLS構成を確認することもできます。

.. code-block:: sql
    :caption: CDB

    SQL> SELECT * FROM CDB_OLS_STATUS;

    NAME                 STATUS               DESCRIPTION                                            CON_ID
    -------------------- -------------------- -------------------------------------------------- ----------
    OLS_CONFIGURE_STATUS FALSE                Determines if OLS is configured                             1
    OLS_ENABLE_STATUS    FALSE                Determines if OLS is enabled                                1
    OLS_CONFIGURE_STATUS FALSE                Determines if OLS is configured                             3
    OLS_ENABLE_STATUS    FALSE                Determines if OLS is enabled                                3


****************************
OLSを有効化する
****************************

SYSユーザーでFREEPDB1にて以下を実行し、OLSを構成します。

.. code-block:: sql

    -- ユーザー名とDBの確認
    SQL> show user con_name
    USER is "SYS"

    CON_NAME
    ------------------------------
    FREEPDB1

    -- OLSを構成する
    SQL> EXEC LBACSYS.CONFIGURE_OLS;

    -- OLSを有効化する
    SQL> EXEC LBACSYS.OLS_ENFORCEMENT.ENABLE_OLS;


再度 DBA_OLS_STATUS を確認し、2つの設定がTRUEになっていることを確認します。


.. code-block:: sql

    SQL> SELECT * FROM DBA_OLS_STATUS;

    NAME                 STATUS               DESCRIPTION
    -------------------- -------------------- --------------------------------------------------
    OLS_CONFIGURE_STATUS TRUE                 Determines if OLS is configured
    OLS_ENABLE_STATUS    TRUE                 Determines if OLS is enabled


設定を完全に反映させるため、FREEPDB1の再起動を行います。

.. code-block:: sql
    :caption: CDBにて実行

    SQL> alter pluggable database freepdb1 close immediate;

    SQL> alter pluggable database freepdb1 open;


****************************
OLS設定に必要な権限の準備
****************************


OLSを操作するためのロール、LBAC_DBAロールを持っているユーザーを確認します。

.. code-block:: sql

    SQL> set markup csv on
    SQL> select * from dba_role_privs where granted_role = 'LBAC_DBA';
    "GRANTEE","GRANTED_ROLE","ADMIN_OPTION","DELEGATE_OPTION","DEFAULT_ROLE","COMMON","INHERITED"
    "SYS"    ,"LBAC_DBA"    ,"YES"         ,"NO"             ,"YES"         ,"YES"   ,"YES"
    "LBACSYS","LBAC_DBA"    ,"YES"         ,"NO"             ,"YES"         ,"YES"   ,"YES"

今後のOLS操作をSYSユーザーで行う場合、SA_SYSDBAパッケージの実行に対して以下のエラーが発生するため、INHERIT PRIVILEGES権限が必要になります。

.. code-block:: sql

    ORA-06598: insufficient INHERIT PRIVILEGES privilege


そのため、SYS ユーザーがLBACSYSの権限を継承できるよう、INHERIT PRIVILEGES を付与します。

.. code-block:: sql

    GRANT INHERIT PRIVILEGES ON USER SYS TO LBACSYS



******************************
OLSポリシーを作成し有効化する
******************************
OLSポリシー（またはポリシーコンテナ）を作成します。

.. ポリシーを作成すると、そのロールが作成され、ユーザーに付与されます。ロール名の形式は、policy_DBAです

.. code-block:: sql

    BEGIN
        SA_SYSDBA.CREATE_POLICY (
            policy_name      => 'OLS_POL_DEMO',
            column_name      => 'OLS_LABEL_DEMO');
    END;
    /

``PL/SQL procedure successfully completed.`` が表示され、無事実行されたことを確認します。


作成したポリシーを有効化します。

.. code-block:: sql
    
    EXEC SA_SYSDBA.ENABLE_POLICY ('OLS_POL_DEMO');


これでOracle Label Securityの準備および設定は完了です。
