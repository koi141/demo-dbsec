############################################
4. ラベルデータの処理とラベル列の非表示
############################################

この手順では、データラベルを使用した操作（絞り込み・ソート）と、ラベル列の非表示オプションを適用する方法を解説します。

.. topic:: 実施内容
    
    + データラベルを使用した操作
    + ラベル列の非表示設定

****************************
データ・ラベルの表示
****************************

見ていただくとわかる通りですが、ラベル列は数値として管理されるため、不等号を使用した条件で絞り込みが可能です。

.. code-block:: sql

    SQL> select JOB_ID, OLS_LABEL_DEMO from HR.JOB_HISTORY_4OLS where OLS_LABEL_DEMO >= 300;

    JOB_ID     OLS_LABEL_DEMO
    ---------- --------------
    AC_ACCOUNT            300
    AC_MGR                400
    MK_REP                300
    SA_REP                300
    SA_MAN                400
    AC_ACCOUNT            300

    6 rows selected.

ラベル列を基準にORDER BYでデータをソートすることも可能です。

.. code-block:: sql

    SQL> select JOB_ID, OLS_LABEL_DEMO from HR.JOB_HISTORY_4OLS order by OLS_LABEL_DEMO;

    JOB_ID     OLS_LABEL_DEMO
    ---------- --------------
    IT_PROG               200
    ST_CLERK              200
    AD_ASST               200
    ST_CLERK              200
    AC_ACCOUNT            300
    SA_REP                300
    MK_REP                300
    AC_ACCOUNT            300
    SA_MAN                400
    AC_MGR                400

    10 rows selected.


****************************
ラベル列の非表示設定
****************************

HIDEオプションを表に適用することで、ポリシーを表す列を非表示にするように選択できます。
しかし、この非表示設定は初回のポリシー適用時のみ可能ですので、すでにポリシーがある場合は一度ポリシーを削除する必要があります。


以下のSQLでポリシーを削除し、OLS_LABEL_DEMO列を表から削除します。

.. code-block:: sql

    BEGIN
        SA_POLICY_ADMIN.REMOVE_TABLE_POLICY (
            policy_name     => 'OLS_POL_DEMO',
            schema_name     => 'HR',
            table_name      => 'JOB_HISTORY_4OLS',
            drop_column     => TRUE
        );
    END;
    /

ラベル列が削除されたことを確認します

.. code-block:: sql

    SQL> desc HR.JOB_HISTORY_4OLS;
    Name                                      Null?    Type
    ----------------------------------------- -------- ----------------------------
    EMPLOYEE_ID                               NOT NULL NUMBER(6)
    START_DATE                                NOT NULL DATE
    END_DATE                                  NOT NULL DATE
    JOB_ID                                    NOT NULL VARCHAR2(10)
    DEPARTMENT_ID                                      NUMBER(4)


HIDEオプションを使用してポリシーを再適用します。

.. code-block:: sql

    BEGIN
        SA_POLICY_ADMIN.APPLY_TABLE_POLICY (
            policy_name    => 'OLS_POL_DEMO',
            schema_name    => 'HR', 
            table_name     => 'JOB_HISTORY_4OLS',
            table_options  => 'READ_CONTROL, HIDE');
    END;
    /

ポリシーの適用後、ラベル列は非表示になっていることが分かります。

.. code-block:: sql
    
    SQL> desc HR.JOB_HISTORY_4OLS;
    Name                                      Null?    Type
    ----------------------------------------- -------- ----------------------------
    EMPLOYEE_ID                               NOT NULL NUMBER(6)
    START_DATE                                NOT NULL DATE
    END_DATE                                  NOT NULL DATE
    JOB_ID                                    NOT NULL VARCHAR2(10)
    DEPARTMENT_ID                                      NUMBER(4)

    SQL> select * from HR.JOB_HISTORY_4OLS;

    EMPLOYEE_ID START_DAT END_DATE  JOB_ID     DEPARTMENT_ID
    ----------- --------- --------- ---------- -------------
            102 13-JAN-11 24-JUL-16 IT_PROG               60
            101 21-SEP-07 27-OCT-11 AC_ACCOUNT           110
    ...


非表示設定でも、明示的に列名を指定すればラベル列を参照することができます。
（ここでは一度ポリシーを削除したため、再適用後のラベル列のデータは空になっています。）


.. code-block:: sql

    SQL> select JOB_ID, OLS_LABEL_DEMO from HR.JOB_HISTORY_4OLS;

    JOB_ID     OLS_LABEL_DEMO
    ---------- --------------
    IT_PROG
    AC_ACCOUNT
    AC_MGR
    MK_REP
    ...


手順2と同様のデータ挿入手順でラベルデータを挿入し直すと、ラベル列のデータも確認できることが分かります。

.. code-block:: sql

    SQL> select JOB_ID, OLS_LABEL_DEMO from HR.JOB_HISTORY_4OLS;

    JOB_ID     OLS_LABEL_DEMO
    ---------- --------------
    IT_PROG               200
    AC_ACCOUNT            300
    AC_MGR                400
    MK_REP                300
    ...

以上でOracle Label Securityの動作確認は終了です。次の手順では構築したOLSの設定を削除していきます。