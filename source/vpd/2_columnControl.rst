###########################
2. VPDで列制御を行う
###########################

where句を使用すると行レベルでの制御になりますが、VPDでは列を制御することも可能です。
この手順では、Virtual Private Database (VPD)を使用して、特定の列へのアクセスを制御します。

.. topic:: 実施内容
    
    + VPD関数を作成する
    + VPDポリシーを作成する
    + HRユーザーで EMPLOYEES 表を確認
    + SALES_APPユーザーで EMPLOYEES 表を確認し、VPDが正しく機能していることを確認
    + 列を非表示ではなくNULL値で表示させる方法



****************************
VPD関数の作成
****************************

列に対するフィルタ条件を定義するVPD関数を作成します。
この例では、SALARY 列をSALES_APPユーザーから非表示にします。

.. code:: sql

    CREATE OR REPLACE FUNCTION hr.get_masking_salary_col( 
        p_schema IN VARCHAR2,
        p_table  IN VARCHAR2
        )
        RETURN VARCHAR2
        IS
            v_predicate VARCHAR2 (400);
        BEGIN
            IF SYS_CONTEXT('USERENV', 'SESSION_USER') = 'SALES_APP' THEN
            v_predicate := '1=2'; -- 常にfalseで列を非表示にする
        END IF;
        RETURN v_predicate;
    END get_masking_salary_col;
    /


実行し、 ``Function created.`` が表示されることを確認します。
ここでは ``v_predicate := '1=2';`` として、常にfalseを渡すことで、SALES_APPユーザーに対しては特定の列が非表示になるようにしています。


****************************
VPDポリシーの作成
****************************
作成した関数を使用し、SALARY 列にポリシーを適用します。


.. code:: sql

    BEGIN
        DBMS_RLS.ADD_POLICY (
            object_schema         => 'HR',
            object_name           => 'EMPLOYEES',
            policy_name           => 'employees_salary_col_vpd_policy',
            function_schema       => 'HR',
            policy_function       => 'get_masking_salary_col',
            sec_relevant_cols     => 'SALARY'
        );
    END;
    /

実行し、 ``PL/SQL procedure successfully completed.`` が表示されることを確認します。


作成したVPDポリシーは ``ALL_POLICIES`` ビューで確認できます。

.. code:: sql

    SQL> set markup csv on
    SQL> select object_owner, object_name, policy_name, function, sel, ins, upd, del, idx, policy_type, common from all_policies where object_owner  = 'HR';
    "OBJECT_OWNER","OBJECT_NAME","POLICY_NAME"                    ,"FUNCTION"              ,"SEL","INS","UPD","DEL","IDX","POLICY_TYPE","COMMON"
    "HR"          ,"EMPLOYEES"  ,"EMPLOYEES_VPD_POLICY"           ,"GET_SALES_PREDICATE"   ,"YES","NO" ,"YES","YES","NO" ,"DYNAMIC"    ,"NO"
    "HR"          ,"EMPLOYEES"  ,"EMPLOYEES_SALARY_COL_VPD_POLICY","GET_MASKING_SALARY_COL","YES","NO" ,"NO" ,"NO" ,"NO" ,"DYNAMIC"    ,"NO"

前手順で作成した ``EMPLOYEES_VPD_POLICY`` に加えてポリシーが作成されたことを確認します。


****************************
HRユーザーで確認
****************************

作成したVPDポリシーが正しく機能しているかを確認します。
念のため、HRユーザーでアクセスし、salary列および107行すべてが表示されることを確かめます。

.. code-block:: sql
    :caption: HRユーザー

    SQL> select first_name, salary from hr.employees;
    "FIRST_NAME","SALARY"
    "Steven",24000
    "Neena",17000
    "Lex",17000
    "Alexander",9000
    ...
    "Michael",13000
    "Pat",6000
    "Susan",6500
    "Hermann",10000
    "Shelley",12008
    "William",8300

    107 rows selected.

    SQL> select * from hr.employees;
    "EMPLOYEE_ID","FIRST_NAME","LAST_NAME","EMAIL","PHONE_NUMBER","HIRE_DATE","JOB_ID","SALARY","COMMISSION_PCT","MANAGER_ID","DEPARTMENT_ID"
    100,"Steven","King","SKING","1.515.555.0100","17-JUN-13","AD_PRES",24000,,,90
    101,"Neena","Yang","NYANG","1.515.555.0101","21-SEP-15","AD_VP",17000,,100,90
    ...


****************************************************************************
SALES_APPユーザーで確認
****************************************************************************


SALES_APPユーザーでは、SALARY 列が含まれるクエリを実行すると、VPDによる制御が適用されます。

.. code-block:: sql
    :caption: salary列を含むクエリ

    SQL> select first_name, salary from hr.employees;

    no rows selected

    SQL> select * from hr.employees;

    no rows selected



.. code-block:: sql
    :caption: salary列を含まないクエリ

    SQL> select first_name from hr.employees;
    "FIRST_NAME"
    "Ellen"
    "Sundar"
    "Amit"
    "Elizabeth"
    "David"
    "Harrison"
    "Gerald"
    ...
    "William"
    "Patrick"
    "Jonathon"
    "Sean"
    "Oliver"
    "Clara"
    "Eleni"

    35 rows selected.


****************************************************************************
列をNULL値で表示する方法（dbms_rls.ALL_ROWS）
****************************************************************************

VPDポリシーを作成する際、デフォルトでは対象列が選択された際にVPDが動作し、先ほどの結果のように値が条件を満たした行しか表示されませんが、
 ``sec_relevant_cols_opt => dbms_rls.ALL_ROWS`` を指定することで、列を非表示ではなくNULL値で表示することができます。


ポリシーの削除と再作成
=======================


既存のVPDポリシーを削除し、新たに作成します。

.. code-block:: sql
    :caption: VPDポリシーを削除
    
    BEGIN
        DBMS_RLS.DROP_POLICY(
            object_schema => 'HR',
            object_name   => 'EMPLOYEES',
            policy_name   => 'employees_salary_col_vpd_policy'
        );
    END;
    /

.. code-block:: sql
    :caption: VPDポリシーを再作成
    
    BEGIN
        DBMS_RLS.ADD_POLICY (
            object_schema         => 'HR',
            object_name           => 'EMPLOYEES',
            policy_name           => 'employees_salary_col_vpd_policy',
            function_schema       => 'HR',
            policy_function       => 'get_masking_salary_col',
            sec_relevant_cols     => 'SALARY',
            sec_relevant_cols_opt => dbms_rls.ALL_ROWS
        );
    END;
    /


SALES_APPユーザーで確認する
================================

SALES_APPユーザーで確認します。
ポリシー再作成後、SALARY 列がNULL値として表示されます。

.. code-block:: sql
    :caption: SALES_APPユーザー

    SQL> select first_name, salary from hr.employees;
    "FIRST_NAME","SALARY"
    "John"      ,
    "Karen"     ,
    "Alberto"   ,
    "Gerald"    ,
    "Eleni"     ,
    "Sean"      ,
    "David"     ,
    "Peter"     ,
    ...
    "Tayler"    ,
    "William"   ,
    "Elizabeth" ,
    "Sundita"   ,
    "Ellen"     ,
    "Alyssa"    ,
    "Jonathon"  ,
    "Jack"      ,
    "Kimberely" ,
    "Charles"   ,

    35 rows selected.


    SQL> select * from hr.employees;
    "EMPLOYEE_ID","FIRST_NAME","LAST_NAME","EMAIL","PHONE_NUMBER","HIRE_DATE","JOB_ID","SALARY","COMMISSION_PCT","MANAGER_ID","DEPARTMENT_ID"
    145,"John","Singh","JSINGH","44.1632.960000","01-OCT-14","SA_MAN",,0,100,80
    146,"Karen","Partners","KPARTNER","44.1632.960001","05-JAN-15","SA_MAN",,0,100,80
    ...


このように、salary列はNULLになっていますが、SALES_APPユーザーでも他の列は通常どおり表示されていることがわかります。

