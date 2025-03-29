############################################
4. 二人制整合性を設定・体験する
############################################

本手順では、Oracle Database Vaultの二人制整合性を使用する方法を説明します。
二人制整合性は、危険を伴う可能性のあるアクションに対して、追加のセキュリティ層を提供するための仕組みです。
例えば、データベースのパッチ適用などの重要な作業を1人のユーザーだけで実施できないようにすることで、
リスクを軽減できます。

.. topic:: 実施内容

    + ユーザーの作成
    + 検証ファンクションの作成
    + ルールおよびルールセットの作成
    + コマンドルールの作成
    + 作成したオブジェクトの削除



********************************
ユーザーの作成
********************************

まず、以下2人のDBユーザーを作成します。接頭辞はTPI（Two-Person Integrity）としています。

+ TPI_BOSS（マネージャ）
+ TPI_USER（作業者）

Oracle Database Vaultが有効化されている環境では、SYSユーザーでは直接ユーザーを作成できないため、
DVを有効化する際に指定した、ユーザー管理用アカウント（C##ACCTMGR）を使用します。


.. code-block:: sql
    :caption: c##dvacctmgrユーザーで実行

    -- ログイン
    SQL> sqlplus c##dvacctmgr/<password>@localhost:1521/FREEPDB1

    -- 2人のユーザーを作成
    SQL> grant create session to tpi_boss identified by <password>;
    SQL> grant create session to tpi_user identified by <password>;


********************************
ファンクションの作成
********************************


次に、DV_OWNERスキーマにTPI_BOSSがログインしているかを確認するファンクションを作成するため、SYSユーザーで権限を付与します。

.. code-block:: sql
    :caption: SYSユーザーで実行

    -- sysユーザーに切り替え
    SQL> conn sys/<password>@localhost:1521/freepdb1 as sysdba

    SQL> grant create procedure to c##dvowner;
    SQL> grant select on v_$session to c##dvowner;

なお、 ``V$SESSION`` は ``V_$SESSION`` のパブリックシノニムのため、権限は ``V_$SESSION`` で指定する必要があることに注意してください。


次に ``TPI_BOSS`` がログインしているかを検証するファンクションを作成します。

.. code-block:: sql
    :caption: C##DVOWNERユーザーで実行

    CREATE OR REPLACE FUNCTION is_boss_logged_in
    RETURN BOOLEAN AS
        v_boss_session NUMBER;
    BEGIN
        SELECT COUNT(*) INTO v_boss_session FROM v$session WHERE username = 'TPI_BOSS';

        IF v_boss_session > 0 THEN 
            RETURN TRUE; -- TPI_BOSSユーザーがログインしていればTRUEを返す
        ELSE
            RETURN FALSE;
        END IF;
    END is_boss_logged_in;
    /

作成したファンクションのEXECUTE権限をDVSYSスキーマに付与します。


.. code-block:: sql
    :caption: C##DVOWNERユーザーで実行
    
    SQL> GRANT EXECUTE ON is_boss_logged_in TO DVSYS;

    Grant succeeded.

.. note::

    Database Vaultのオブジェクトおよびファンクションは、主にDVSYSスキーマに格納されます。（また、他にDVFスキーマがあります）
    そのため、2人制ルールを使用するに関わらずユーザー定義の関数を使用する際は、DVSYSスキーマへのEXECUTE権限が必要になることに注意してください。
    もちろん代わりにファンクションをDVSYSスキーマに作成することもできますが、このスキーマはデフォルトでロックされており、通常ロックしたままのスキーマ専用アカウントとして扱われます。
    （参考： `Oracle Database Vault DVSYSおよびDVFスキーマ <https://docs.oracle.com/cd/F82042_01/dvadm/introduction-to-oracle-database-vault.html#GUID-78C38076-42E7-463A-B111-214F6958A425>`__ ）


********************************
ルールの作成
********************************

作成したファンクションを指定し、ルールを作成します。

.. code-block:: sql
    :caption: C##DVOWNERユーザーで実行

    BEGIN
        DBMS_MACADM.CREATE_RULE(
            rule_name => 'Rule to check tpi_Boss Login',
            rule_expr => 'SYS_CONTEXT(''USERENV'',''SESSION_USER'') = ''TPI_USER'' AND C##DVOWNER.IS_BOSS_LOGGED_IN = TRUE'
        );
    END;
    /

なお、このルールを作成する前に、dual表を用いて以下のように正しく条件判定が出来ているかを確認するといいと思います。

.. code-block::

    select SYS_CONTEXT('USERENV','SESSION_USER') = 'TPI_USER' AND C##DVOWNER.IS_BOSS_LOGGED_IN = TRUE from dual;


このままでは、Bossユーザー含め、誰もルールに適用しないため、どのユーザーもログインできません。
そのため、TPI_USERユーザー以外はTPI_BOSSのログインがなくともログインできるように設定します。

.. code-block:: sql
    :caption: C##DVOWNERユーザーで実行

    BEGIN
        DBMS_MACADM.CREATE_RULE(
            rule_name => 'Rule to allow Other Users Access',
            rule_expr => 'SYS_CONTEXT(''USERENV'',''SESSION_USER'') != ''TPI_USER'' '
        );
    END;
    /

*********************************************
ルール・セットの作成とルールの追加
*********************************************

ルール・セットを作成します。

.. code-block:: sql
    :caption: C##DVOWNERユーザーで実行

    BEGIN
        DBMS_MACADM.CREATE_RULE_SET(
            rule_set_name    => 'Ruleset for Dual Connect',
            description      => 'Ensures both the tpi_Boss and tpi_User are logged in before allowing access.',
            enabled          => DBMS_MACUTL.G_YES,                 -- (*)
            eval_options     => DBMS_MACUTL.G_RULESET_EVAL_ANY, -- ルールセットのいずれかがTrueになることで有効化される
            fail_message     => 'DV_Error: Access restricted unless both tpi_Boss is logged in.',
            fail_code        => 20000,
            handler_options  => DBMS_MACUTL.G_RULESET_HANDLER_OFF, -- (*)
            handler          => '',
            is_static        => FALSE,                             -- (*)
            scope            => DBMS_MACUTL.G_SCOPE_LOCAL
        );
    END;
    /

作成したルール・セットにルールを追加します

.. code-block:: sql
    :caption: C##DVOWNERユーザーで実行
    
    BEGIN
        DBMS_MACADM.ADD_RULE_TO_RULE_SET(
            rule_set_name  => 'Ruleset for Dual Connect',
            rule_name      => 'Rule to check tpi_Boss Login',
            rule_order     => 1,
            enabled        => DBMS_MACUTL.G_YES     -- (*)
        );
    END;
    /   

    BEGIN
        DBMS_MACADM.ADD_RULE_TO_RULE_SET(
            rule_set_name  => 'Ruleset for Dual Connect',
            rule_name      => 'Rule to allow Other Users Access',
            rule_order     => 1,
            enabled        => DBMS_MACUTL.G_YES     -- (*)
        );
    END;
    /



********************************
コマンド・ルールの作成
********************************

TPI_BOSSがログインしている場合のみ、TPI_USERがログインできるようにコマンド・ルールを作成します。

.. code-block:: sql
    :caption: C##DVOWNERユーザーで実行

    BEGIN
        DBMS_MACADM.CREATE_COMMAND_RULE(
            command            => 'CONNECT',
            rule_set_name      => 'Ruleset for Dual Connect',
            object_owner       => '%',
            object_name        => '%',
            enabled            => DBMS_MACUTL.G_YES
        );
    END;
    /

    COMMIT;


********************************
二人制整合性を体験する
********************************

では、準備ができたのでTPI_USERが承認（TPI_BOSSのログイン）によって正しくログインできるかを確認します。

2つの端末を用意し、それぞれでTPI_USERとTPI_BOSSでログインします。

まず、TPI_USERユーザーでログインしようとするとできないことを確認します。

.. code-block:: sql
    :caption: 端末Ａにて

    SQL> conn tpi_user/<password>@localhost:1521/FREEPDB1
    ERROR:
    ORA-47306: 20000: DV_Error: Access restricted unless both Boss is logged in.
    Help: https://docs.oracle.com/error-help/db/ora-47306/

    -- 他のユーザーではログインできる(以下はAPPユーザーを作成している例)
    SQL> conn app/<password>@localhost:1521/FREEPDB1
    Connected.

設定したエラーメッセージが表示され、ルールが正しく検知できていることが確認できます。

次にtpi_bossでログインし、その状態を維持します。

.. code-block:: sql
    :caption: 端末Bにて

    SQL> conn tpi_boss/<password>@localhost:1521/FREEPDB1
    Connected.

再びtpi_userユーザーでログインを行います。

.. code-block:: sql
    :caption: 端末Ａにて

    SQL> conn tpi_user/<password>@localhost:1521/FREEPDB1
    Connected.

この動作により、TPI_BOSSがログインしている間のみTPI_USERがログインできることが確認できました。


********************************
作成したオブジェクトの掃除
********************************

最後に作成したオブジェクトを削除します。

.. code-block:: sql
    :caption: c##dvacctmgrユーザーで実行

    DROP USER tpi_boss;
    DROP USER tpi_user;


.. code-block:: sql
    :caption: sysユーザーで実行

    REVOKE CREATE PROCEDURE FROM c##dvowner;
    REVOKE SELECT ON V_$SESSION FROM c##dvowner;


.. code-block:: sql
    :caption: C##DVOWNERで実行

    DROP FUNCTION is_boss_logged_in;

    EXEC DBMS_MACADM.DELETE_RULE_FROM_RULE_SET(rule_set_name => 'Ruleset for Dual Connect', rule_name => 'Rule to check tpi_Boss Login');
    EXEC DBMS_MACADM.DELETE_RULE_FROM_RULE_SET(rule_set_name => 'Ruleset for Dual Connect', rule_name => 'Rule to allow Other Users Access');

    EXEC DBMS_MACADM.DELETE_COMMAND_RULE(command => 'CONNECT', object_owner => '%', object_name => '%');
    EXEC DBMS_MACADM.DELETE_RULE('Rule to check tpi_Boss Login');
    EXEC DBMS_MACADM.DELETE_RULE('Rule to allow Other Users Access');

    EXEC DBMS_MACADM.DELETE_RULE_SET('Ruleset for Dual Connect');

    COMMIT;


以上で、二人制整合性を用いたセキュリティ設定の解説は終了です。