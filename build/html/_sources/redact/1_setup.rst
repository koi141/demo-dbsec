##########################################
1. Data Redactionの準備
##########################################

**実施内容**

+ 結果の比較用ユーザーとしてSALES_APPユーザーを作成する
+ リダクションポリシーを作成する


*****************************************
SALES_APPユーザーを作成
*****************************************

HRユーザーとリダクションの結果を比較するために、マスキングされたデータを返すユーザーとしてSALES_APP ユーザーを作成します。


まず、SALES_APP ユーザーを作成します。

.. code:: sql

    CREATE USER SALES_APP IDENTIFIED BY <password> DEFAULT TABLESPACE USERS TEMPORARY TABLESPACE TEMP;
    CREATE USER SALES_APP IDENTIFIED BY Welcome1#Welcom1# DEFAULT TABLESPACE USERS TEMPORARY TABLESPACE TEMP;


以下は実行例です。証跡としてパスワードが残らないよう、2つのコマンドに分けて設定しています。

.. code:: sql

    SQL> create user sales_app
    2     default tablespace users
    3     temporary tablespace temp;

    -- passwordを設定
    SQL> password sales_app
    Changing password for sales_app
    New password: <パスワードを入力>
    Retype new password: <パスワードを再入力>
    Password changed


次に、SALES_APP ユーザーにセッション作成権限を付与します。

.. code:: sql

    SQL> grant create session to sales_app;


| さらに、HRスキーマに対して SELECT 権限を付与します。
| スキーマ単位で権限付与する方法は23aiの新機能となっており、これにより、SALES_APP ユーザーは HR スキーマのテーブルに対してデータを参照できるようになります。

.. code:: sql

    -- 23aiの新機能、スキーマ権限
    SQL> grant select any table on schema HR to sales_app;



*****************************************
リダクションポリシーを作成する
*****************************************

| 次に、SALES_APP ユーザーには EMPLOYEES テーブルの ``SALARY`` 列と ``COMMISSION_PCT`` 列をマスキングするポリシーを作成します。
| 営業が使用するアプリケーションでは給与と手数料の値は使用しないため閲覧は必要ない、という架空の設定です。

一度に一つの列にしかリダクションポリシーを適用できないため、 ``SALARY`` 列と ``COMMISSION_PCT`` 列に別々にポリシーを適用します。

まず、SALARY 列に対してリダクションポリシーを作成します。以下のコマンドを実行します：

.. code:: sql

    BEGIN
        DBMS_REDACT.ADD_POLICY(
            object_schema       => 'HR',
            object_name         => 'EMPLOYEES',
            column_name         => 'SALARY',
            policy_name         => 'POL_REDCT_EMPLOYEES_SALARY',
            function_type       => DBMS_REDACTION.CONSTANT,
            function_parameters => 0,
            expression          => 'SYS_CONTEXT(''USERENV'', ''SESSION_USER'') = ''SALES_APP'''
        );
    END;
    /

次に、先に作成したPOL_REDCT_EMPLOYEES_SALARYポリシーに新たに列を追加する形で ``COMMISSION_PCT`` 列に対してリダクションポリシーを追加します。

.. code:: sql

    BEGIN
        DBMS_REDACT.ADD_POLICY(
            object_schema  => 'HR',
            object_name    => 'EMPLOYEES',
            column_name    => 'SALARY',
            policy_name    => 'POL_REDCT_EMPLOYEES_SALARY',
            function_type  => DBMS_REDACT.FULL,
            expression     => 'SYS_CONTEXT(''USERENV'', ''SESSION_USER'') = ''SALES_APP'''
        );
    END;
    /

リダクションポリシーが正常に作成されたかを確認します。

.. code:: sql

    -- ポリシーの作成を確認
    SQL> select * from redaction_policies;
    "OBJECT_OWNER","OBJECT_NAME","POLICY_NAME"               ,"EXPRESSION"                                          ,"ENABLE","POLICY_DESCRIPTION"
    "HR"          ,"EMPLOYEES"  ,"POL_REDCT_EMPLOYEES_SALARY","SYS_CONTEXT('USERENV', 'SESSION_USER') = 'SALES_APP'","YES"   ,

    -- リダクション対象の列を確認
    SQL> select object_owner, object_name, column_name, function_type from redaction_columns;
    "OBJECT_OWNER","OBJECT_NAME","COLUMN_NAME"   ,"FUNCTION_TYPE"
    "HR"          ,"EMPLOYEES"  ,"SALARY"        ,"FULL REDACTION"
    "HR"          ,"EMPLOYEES"  ,"COMMISSION_PCT","FULL REDACTION"


これで、SALES_APPユーザーが ``EMPLOYEES`` テーブルの ``SALARY`` 列と ``COMMISSION_PCT`` 列へのアクセスに対して、リダクションが適用されるようになりました。





