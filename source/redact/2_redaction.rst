##########################################
2. Redactionを体験する
##########################################

**実施内容**

+ HRユーザーとSALES_APPユーザーで実行結果が異なることを確認する


*****************************************
HRユーザーで接続
*****************************************

| まず、HRユーザーとしてデータベースに接続し、 ``EMPLOYEES`` テーブルの ``first_name`` 、 ``salary`` 、 ``commission_pct`` 列を取得します。（実行結果は一部を抜粋しています）


.. code-block:: sql

    $ sqlplus hr/<password>@localhost:1521/freepdb1

    SQL> select first_name, salary, commission_pct from employees;
    ...
    FIRST_NAME               SALARY COMMISSION_PCT
    -------------------- ---------- --------------
    Peter                      2500
    John                      14000             .4
    Karen                     13500             .3
    Alberto                   12000             .3
    Gerald                    11000             .3
    Eleni                     10500             .2
    ...
    107 rows selected.

``salary`` 列と ``commission_pct`` 列のデータがそのまま表示され、HR ユーザーは元のデータにアクセスできていることがわかります。


*****************************************
SALES_APPユーザーで接続
*****************************************

| 次に、SALES_APPユーザーとしてデータベースに接続し、同じクエリを実行します。
| このユーザーにはリダクションポリシーが適用されているため、マスキングされたデータが返るはずです。（実行結果は一部を抜粋しています）


.. code-block:: sql

    $ sqlplus sales_app/<password>@localhost:1521/freepdb1

    select first_name, salary, commission_pct from hr.employees;
    ...
    FIRST_NAME               SALARY COMMISSION_PCT
    -------------------- ---------- --------------
    Peter                         0
    John                          0              0
    Karen                         0              0
    Alberto                       0              0
    Gerald                        0              0
    Eleni                         0              0
    ...
    107 rows selected.

| SALES_APPユーザーでは、 ``salary`` 列と ``commission_pct`` 列の値が全て 0 で表示されており、リダクションポリシーが正しく適用されていることが確認できます。
| これにより、SALES_APPユーザーは給与や手数料の値を閲覧することができなくなっています。