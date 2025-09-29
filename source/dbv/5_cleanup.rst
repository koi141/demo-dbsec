############################################
5. Database Vaultを無効化する
############################################

Database Vaultで設定したレルムおよび関連する認可、オブジェクト登録を削除し、Database Vault自体を無効化します。

.. topic:: 実施内容

    + レルムの無効化
    + レルム認可の削除
    + レルムからのオブジェクト削除
    + レルムの削除
    + Database Vault の無効化



********************************
レルムの無効化
********************************
作成したレルム (Realm for demo) を無効化します。

.. code-block:: sql
    :caption: C##DVOWNERユーザー

    BEGIN
        DBMS_MACADM.UPDATE_REALM(
            realm_name     => 'Realm for demo',
            enabled        => DBMS_MACUTL.G_NO);
    END;
    /


********************************
レルム認可の削除
********************************
レルム内のオブジェクトへのアクセス権を付与した認可を削除します。

.. code-block:: sql
    :caption: C##DVOWNERユーザー

    -- HRユーザーの認可を削除
    BEGIN
        DBMS_MACADM.DELETE_AUTH_FROM_REALM(
            realm_name    => 'Realm for demo',
            grantee       => 'HR',
            auth_scope    => DBMS_MACUTL.G_SCOPE_LOCAL);
    END;
    /

    -- SALES_APPユーザーの認可を削除
    BEGIN
        DBMS_MACADM.DELETE_AUTH_FROM_REALM(
            realm_name    => 'Realm for demo',
            grantee       => 'SALES_APP',
            auth_scope    => DBMS_MACUTL.G_SCOPE_LOCAL);
    END;
    /

    -- APPユーザーの認可を削除
    BEGIN
        DBMS_MACADM.DELETE_AUTH_FROM_REALM(
            realm_name    => 'Realm for demo',
            grantee       => 'APP',
            auth_scope    => DBMS_MACUTL.G_SCOPE_LOCAL);
    END;
    /

完全に削除されたことを確認します。

.. code-block:: sql
    :caption: C##DVOWNERユーザー

    SQL> select realm_name, grantee from DVSYS.DBA_DV_REALM_AUTH where realm_name = 'Realm for demo';

    no rows selected


********************************
レルムからオブジェクトを削除する
********************************
レルムに登録されていたオブジェクト (COUNTRIES表とREGIONS表) を削除します。

.. code-block:: sql
    :caption: C##DVOWNERユーザー

    -- COUNTRIES表をレルムから削除
    BEGIN
        DBMS_MACADM.DELETE_OBJECT_FROM_REALM(
            realm_name   => 'Realm for demo',
            object_owner => 'HR',
            object_name  => 'COUNTRIES',
            object_type  => 'TABLE');
    END;
    /

    -- REGIONS表をレルムから削除
    BEGIN
        DBMS_MACADM.DELETE_OBJECT_FROM_REALM(
            realm_name   => 'Realm for demo',
            object_owner => 'HR',
            object_name  => 'REGIONS',
            object_type  => 'TABLE');
    END;
    /

オブジェクトがレルムから削除されたことを確認します。

.. code-block:: sql
    :caption: C##DVOWNERユーザー

    SQL> select REALM_NAME, OWNER, OBJECT_NAME, OBJECT_TYPE from DVSYS.DBA_DV_REALM_OBJECT where realm_name = 'Realm for demo';

    no rows selected


********************************
レルムの削除
********************************

最後にレルム自体を削除します。

.. code-block:: sql
    :caption: C##DVOWNERユーザー

    BEGIN
        DBMS_MACADM.DELETE_REALM(realm_name  => 'Realm for demo'); 
    END;
    /

********************************
ルール・セットの削除
********************************
ルール・セットを削除します。

.. code-block:: sql
    :caption: C##DVOWNERユーザー

    EXEC DBMS_MACADM.DELETE_RULE_SET('Ruleset for APP'); 

********************************
Database Vaultの無効化
********************************


.. code-block:: sql
    :caption: C##DVOWNERユーザー

    EXEC DBMS_MACADM.DISABLE_DV;

    -- 無効化されたことを確認する
    SQL> SELECT * FROM CDB_DV_STATUS;
    "NAME"               ,"STATUS"        ,"CON_ID"
    "DV_CONFIGURE_STATUS","TRUE"          ,3
    "DV_ENABLE_STATUS"   ,"FALSE"         ,3
    "DV_APP_PROTECTION"  ,"NOT CONFIGURED",3

CDBに接続し、PDBを再起動します。

.. code-block:: sql
    :caption: C##DVOWNERユーザー

    SQL> alter pluggable database freepdb1 close immediate;

    SQL> alter pluggable database freepdb1 open;

SYSユーザーでDBユーザーが作成できるようになり、DB Vaultが無効化されたことが分かります。

.. code-block:: sql
    :caption: SYSユーザー

    SQL> show user con_name
    USER is "SYS"

    CON_NAME
    ------------------------------
    FREEPDB1

    SQL> create user test;

    User created.

    SQL> drop user test;

    User dropped.


以上でDatabase Vaultのデモは終了です。