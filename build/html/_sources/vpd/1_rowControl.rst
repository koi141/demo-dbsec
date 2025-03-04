###########################
1. VPDで行制御を行う
###########################

この手順では、Virtual Private Database (VPD)を使用して、Oracle Databaseの行レベルアクセス制御を実装します。

**実施内容**

+ VPD関数を作成する
+ VPDポリシーを作成する
+ HRユーザーでEMMPLOYEES表を確認する
+ SALES_APPユーザーでEMMPLOYEES表を確認し、VPDが機能していることを確かめる


****************************
VPD関数の作成
****************************

最初に、アクセス制御の条件を定義するVPD関数を作成します。この関数は以下の動作を行います。

+ SALES_APPユーザーの場合: JOB_IDがSA_で始まる行のみを表示
+ その他のユーザーの場合: すべての行を表示

VPDポリシーの対象列としては、外部キーであるJOB_IDを使用します。

.. code:: sql

    CREATE OR REPLACE FUNCTION hr.get_sales_predicate( 
        p_schema IN VARCHAR2,
        p_table  IN VARCHAR2
        )
        RETURN VARCHAR2
        IS
            v_predicate VARCHAR2 (400);
        BEGIN
            IF SYS_CONTEXT('USERENV', 'SESSION_USER') = 'SALES_APP' THEN  
                v_predicate := 'JOB_ID LIKE ''SA_%''';  -- SALESチームのみ表示
            ELSE
                v_predicate := '1=1'; -- その他のユーザーは全件表示
            END IF;
            RETURN v_predicate;
        END get_sales_predicate;
    /

実行し、 ``Function created.`` が表示されることを確認します。

****************************
VPDポリシーの作成
****************************

作成したVPD関数を指定してVPDポリシーを作成していきます。
実行するADD_POLICYプロシージャのパラメータについての詳細は `こちらから <https://docs.oracle.com/cd/F19136_01/arpls/DBMS_RLS.html#GUID-1E528A51-DE53-4961-8770-C53924E427CC>`__ ご確認ください。

.. code:: sql

    BEGIN
    DBMS_RLS.ADD_POLICY (
        object_schema   => 'HR',
        object_name     => 'EMPLOYEES',
        policy_name     => 'employees_vpd_policy',
        function_schema => 'HR',
        policy_function => 'get_sales_predicate'
        );
    END;
    /

実行し、 ``PL/SQL procedure successfully completed.`` が表示されることを確認します。

作成したVPDポリシーは ``ALL_POLICIES`` ディクショナリビューから確認できます。

.. code:: sql

    SQL> select object_owner, object_name, policy_name, function, sel, ins, upd, del, idx, policy_type, common from all_policies where object_owner  = 'HR';
    "OBJECT_OWNER","OBJECT_NAME","POLICY_NAME"         ,"FUNCTION"           ,"SEL","INS","UPD","DEL","IDX","POLICY_TYPE","COMMON"
    "HR"          ,"EMPLOYEES"  ,"EMPLOYEES_VPD_POLICY","GET_SALES_PREDICATE","YES","NO" ,"YES","YES","NO" ,"DYNAMIC"    ,"NO"


各列の説明は以下のとおりです。

===============  ============================================================
列名              説明 
===============  ============================================================
OBJECT_OWNER     対象オブジェクトの所有者
OBJECT_NAME      対象オブジェクトの名前
POLICY_NAME      ポリシー名
FUNCTION         ポリシー関数
SEL              SELECT文に適用されるか
INS              INSERT文に適用されるか
UPD              UPDATE文に適用されるか
DEL              DELETE文に適用されるか
POLICY_TYPE      ポリシーのタイプ
COMMON           ポリシーの適用範囲 (YES: すべてのPDB、NO: ローカルPDBのみ)
===============  ============================================================



****************************
HRユーザーで確認を行う
****************************

HRユーザーで対象の表を参照し、作成したVPDポリシーが正しく機能しているかを確認します。

.. code-block:: sql
    :caption: HRユーザーで実行

    SQL> set markup csv on
    SQL> select employee_id, first_name, salary, job_id from hr.employees;
    "EMPLOYEE_ID","FIRST_NAME","SALARY","JOB_ID"
    100          ,"Steven"    ,24000   ,"AD_PRES"
    101          ,"Neena"     ,17000   ,"AD_VP"
    102          ,"Lex"       ,17000   ,"AD_VP"
    103          ,"Alexander" ,9000    ,"IT_PROG"
    ...
    203          ,"Susan"     ,6500    ,"HR_REP"
    204          ,"Hermann"   ,10000   ,"PR_REP"
    205          ,"Shelley"   ,12008   ,"AC_MGR"
    206          ,"William"   ,8300    ,"AC_ACCOUNT"

107 rows selected.

HRユーザーでアクセスすると、VPD関数が'1=1'を返すため、全てのデータ（107行）が表示されます。

****************************
SALES_APPユーザーで確認
****************************
SALES_APPユーザーで同様のSQL文を実行し、VPDが動作していることを確認します。

.. code-block:: sql
    :caption: SALES_APPユーザーで実行

    SQL> select employee_id, first_name, salary, job_id from hr.employees;
    "EMPLOYEE_ID","FIRST_NAME","SALARY","JOB_ID"
    145          ,"John"      ,0       ,"SA_MAN"
    146          ,"Karen"     ,0       ,"SA_MAN"
    147          ,"Alberto"   ,0       ,"SA_MAN"
    148          ,"Gerald"    ,0       ,"SA_MAN"
    149          ,"Eleni"     ,0       ,"SA_MAN"
    ...
    172          ,"Elizabeth" ,0       ,"SA_REP"
    173          ,"Sundita"   ,0       ,"SA_REP"
    174          ,"Ellen"     ,0       ,"SA_REP"
    175          ,"Alyssa"    ,0       ,"SA_REP"
    176          ,"Jonathon"  ,0       ,"SA_REP"
    177          ,"Jack"      ,0       ,"SA_REP"
    178          ,"Kimberely" ,0       ,"SA_REP"
    179          ,"Charles"   ,0       ,"SA_REP"

    35 rows selected.

この場合はVPDが動作し、where句に「JOB_ID LIKE 'SA_%'」が追加されるため、JOB_IDが'SA_'で始まる35行のみが表示されることが分かります。

次のステップでは、VPDを使用した列レベル制御を行ってみます。