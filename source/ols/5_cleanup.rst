############################################
5. OLS設定の削除手順
############################################

以下の手順では、OLSのデモで設定したポリシー、ラベル、レベル、ユーザーアクセス権を削除し、そしてOLSを無効化します。

**実施内容**

+ ポリシーの無効化とスキーマからの削除
+ ユーザーラベルの削除
+ ラベルの削除
+ レベルの削除
+ ポリシーの削除
+ OLSの無効化とPDBの再起動


*******************************************
 ポリシーの無効化とスキーマからの削除
*******************************************
指定したポリシーをスキーマから削除します。

スキーマ内のすべての表からポリシーを削除し、ラベル列もオプションで削除します。


.. code-block:: sql

    BEGIN
        SA_POLICY_ADMIN.REMOVE_SCHEMA_POLICY(
            policy_name      => 'OLS_POL_DEMO',
            schema_name      => 'HR',
            drop_column      => TRUE);
    END;
    /


*************************
ユーザーラベルの削除
*************************

指定したユーザーから、OLSポリシーに関連するすべての認可を削除します。

.. code-block:: sql

    BEGIN
        SA_USER_ADMIN.DROP_USER_ACCESS (
            policy_name   => 'OLS_POL_DEMO',
            user_name     => 'HR'); 

        SA_USER_ADMIN.DROP_USER_ACCESS (
            policy_name   => 'OLS_POL_DEMO',
            user_name     => 'SALES_APP'); 
    END;
    /

*************************
ラベルの削除
*************************

ポリシー内で作成されたラベルを削除します。

.. code-block:: sql

    BEGIN
        SA_LABEL_ADMIN.DROP_LABEL (
            policy_name  => 'OLS_POL_DEMO',
            label_value  => 'SENS');

        SA_LABEL_ADMIN.DROP_LABEL (
            policy_name  => 'OLS_POL_DEMO',
            label_value  => 'CONF');

        SA_LABEL_ADMIN.DROP_LABEL (
            policy_name  => 'OLS_POL_DEMO',
            label_value  => 'INTL');
    END;
    /


*************************
レベルの削除
*************************

ポリシーで使用されているレベルを削除します。ただし、データまたはユーザーラベルで使用中のレベルは削除できません。

.. code-block:: sql

    BEGIN
        SA_COMPONENTS.DROP_LEVEL (
        policy_name => 'OLS_POL_DEMO',
        short_name  => 'SENS');

        SA_COMPONENTS.DROP_LEVEL (
        policy_name => 'OLS_POL_DEMO',
        short_name  => 'CONF');

        SA_COMPONENTS.DROP_LEVEL (
        policy_name => 'OLS_POL_DEMO',
        short_name  => 'INTL');
    END;
    /


*************************
ポリシーの削除
*************************

ポリシーを削除します。削除の前に無効化する必要はありません。

.. code-block:: sql

    BEGIN
        SA_SYSDBA.DROP_POLICY ( 
            policy_name  => 'OLS_POL_DEMO',
            drop_column  => True);
    END;
    /




********************************
OLSの無効化とPDBの再起動
********************************

OLSポリシーの施行を無効にします。
ただし、Database Vaultを使用している場合は無効化しないでください。

.. code-block:: sql
    
    EXEC LBACSYS.OLS_ENFORCEMENT.DISABLE_OLS;

状態を確認します。

.. code-block:: sql

    SQL> col status for a20
    SQL> col description for a50
    SQL> set lines 100
    SQL> SELECT * FROM DBA_OLS_STATUS;

    NAME                 STATUS               DESCRIPTION
    -------------------- -------------------- --------------------------------------------------
    OLS_CONFIGURE_STATUS TRUE                 Determines if OLS is configured
    OLS_ENABLE_STATUS    FALSE                Determines if OLS is enabled


FALSEとなり、無効化されたことが分かります。

設定を完全に反映させるためにPDBの再起動を行います。

.. code-block:: sql
    :caption: CDBにて実行

    SQL> alter pluggable database freepdb1 close immediate;

    SQL> alter pluggable database freepdb1 open;



以上でOracle Label Securityのデモは終了です。